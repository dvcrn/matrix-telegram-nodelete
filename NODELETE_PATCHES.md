# Matrix Bridge No-Delete Patch Guide

This document explains the modifications made to a mautrix bridge to prevent message deletion propagation from the remote platform to Matrix. When a message is deleted on the remote platform side, instead of deleting it in the Matrix room, the bridge posts a notice (as a reply to the original message) indicating that a deletion was attempted.

Deletions initiated from the Matrix side still work normally and propagate to the remote platform. Capabilities are left unchanged.

This guide is written generically so the same pattern can be applied to any mautrix bridgev2-based bridge.

## Overview of Changes

Only one file is modified:

- **The remote-to-Matrix event handler** (e.g., `handletelegram.go`, `handlemeta.go`) — intercepts incoming delete events and converts them into notice messages instead of propagating the deletion

## Detailed Changes

### Intercept Remote Delete Events (`handle<platform>.go`)

Find the function that handles incoming message deletion events from the remote platform. In a bridgev2 bridge, this typically queues a `simplevent.MessageRemove` event.

**Before:**

```go
func (tc *Client) onDeleteMessages(ctx context.Context, ...) error {
    // ... resolve portal key from message ID ...

    res := tc.main.Bridge.QueueRemoteEvent(tc.userLogin, &simplevent.MessageRemove{
        EventMeta: simplevent.EventMeta{
            Type:      bridgev2.RemoteEventMessageRemove,
            PortalKey: portalKey,
        },
        TargetMessage:   messageID,
        HidePlaceholder: true,
    })
    return resultToError(res)
}
```

**After:**

Replace the `MessageRemove` event with a `simplevent.Message` that posts a notice as a reply to the original message:

```go
func (tc *Client) onDeleteMessages(ctx context.Context, ...) error {
    // ... resolve portal key from message ID (keep this logic unchanged) ...

    // Log the interception
    zerolog.Ctx(ctx).Info().
        Str("message_id", string(messageID)).
        Msg("Intercepted message delete attempt, sending notice instead.")

    // Generate a unique ID for the notice message
    noticeID := networkid.MessageID(
        fmt.Sprintf("delete_notice_%s_%d", messageID, time.Now().UnixMilli()),
    )

    // Queue a new message event instead of a delete event
    res := tc.main.Bridge.QueueRemoteEvent(tc.userLogin, &simplevent.Message[any]{
        EventMeta: simplevent.EventMeta{
            Type:      bridgev2.RemoteEventMessage,
            PortalKey: portalKey,
            Sender:    bridgev2.EventSender{},
            Timestamp: time.Now(),
        },
        ID: noticeID,
        ConvertMessageFunc: func(
            ctx context.Context,
            portal *bridgev2.Portal,
            intent bridgev2.MatrixAPI,
            data any,
        ) (*bridgev2.ConvertedMessage, error) {
            return &bridgev2.ConvertedMessage{
                // Reply to the original message that was targeted for deletion
                ReplyTo: &networkid.MessageOptionalPartID{MessageID: messageID},
                Parts: []*bridgev2.ConvertedMessagePart{{
                    Type: event.EventMessage,
                    Content: &event.MessageEventContent{
                        MsgType: event.MsgText,
                        Body:    fmt.Sprintf(
                            "🚮 Message deletion attempted (ID: %s)", messageID,
                        ),
                    },
                }},
            }, nil
        },
    })
    return resultToError(res)
}
```

**Key points:**

- The portal key resolution logic (looking up the message in the database, cache, etc.) stays unchanged — you still need to know which room the notice goes to.
- `bridgev2.EventSender{}` (empty sender) means the message appears as coming from the bridge bot itself, not from any specific user.
- `ReplyTo` makes the notice a reply to the original message, providing clear context about which message was targeted.
- The `noticeID` must be unique per deletion attempt. Using a combination of the original message ID and current timestamp ensures uniqueness.
- `event.MsgText` is used (not `event.MsgNotice`) so the message is visible in all Matrix clients without special settings.
- `Timestamp: time.Now()` must be set explicitly — without it, the event defaults to Go's zero time (year 1) and gets sorted incorrectly in Matrix room history.
- No changes are needed to `HandleMatrixMessageRemove` or capabilities — deletions from the Matrix side should still propagate normally to the remote platform.

## Required Imports

The following imports are needed in the event handler file:

```go
import (
    "fmt"
    "time"

    "github.com/rs/zerolog"
    "maunium.net/go/mautrix/bridgev2"
    "maunium.net/go/mautrix/bridgev2/networkid"
    "maunium.net/go/mautrix/bridgev2/simplevent"
    "maunium.net/go/mautrix/event"
)
```

Most of these are typically already imported in a bridge's event handler file.

## Alternative Approach: Custom Event Type

Instead of using `simplevent.Message` with a closure, you can define a dedicated struct that implements the `bridgev2.RemoteMessage` interface. This is a cleaner approach if you want more control:

```go
type DeleteNoticeEvent struct {
    portalKey        networkid.PortalKey
    deletedMessageID string
    timestamp        time.Time
}

// Implement: GetType, GetPortalKey, AddLogContext, GetSender, GetID, GetTimestamp, ConvertMessage
```

The `simplevent.Message[any]` approach used here is simpler and requires no new types, making it a smaller diff that's easier to maintain when rebasing on upstream updates.

## Testing

After applying the changes:

1. Build the bridge: `go build ./...`
2. Run `go vet ./...` to check for unused variables or imports
3. Start the bridge and test:
   - Delete a message on the remote platform side. Verify that in Matrix the original message stays and a reply appears saying "🚮 Message deletion attempted (ID: ...)"
   - Delete/redact a message in Matrix. Verify it still propagates and deletes on the remote platform as normal.

## Maintenance Notes

When rebasing on upstream bridge updates:
- Check if the delete handler function signature changed
- The `simplevent.Message` API is stable in bridgev2, so the notice pattern should continue to work across versions
