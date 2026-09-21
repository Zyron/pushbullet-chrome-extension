# Duplicate Push Investigation Report

## Status

The duplicate-tab race is fixed in the local working tree and the latest
service-worker capture shows one tab opened for one incoming link. The same
capture uncovered a separate repeated-notification issue, which is also fixed
locally. These changes have not yet been committed or pushed.

## What was observed

- Chrome received several `tickle` events with `subtype: 'push'` for the same
  Pushbullet activity.
- The active service worker is the current checkout, rather than the old
  Downloads archive. Its log line numbers match the current `background.js`
  startup and WebSocket handlers.
- For the final link push in the capture, there is exactly one
  `Auto-opening link:` entry, followed by persistence of its timestamp and
  push identifier.
- Earlier in the capture, the same notification identifier was created
  repeatedly after repeated tickles. This affected notifications, not tab
  opening.

Routine messages such as `nop`, `No popup open to receive push updates`, and
unknown Pushbullet event types such as `mirror`, `dismissal`, `ping`, and
`pong` do not create link tabs.

## Root causes

### Concurrent auto-open paths

One incoming push can be processed by more than one event path: a WebSocket
tickle, a direct WebSocket push, a scheduled refresh, or a session refresh.
Previously, each path could check whether a push had been opened before a
different path recorded it as opened. Both paths could therefore create a tab.

### Repeated notifications

The tickle handler fetches recent pushes and displayed a notification for the
newest result on every tickle. It did not remember that it had already shown a
notification for that push identifier.

## Fixes in the working tree

### Serialized auto-open processing

All calls to `processPushesForAutoOpen` now share one promise queue. A caller
must finish opening and recording a push before the next caller can test that
same push. This preserves chronological processing while preventing the
check-then-open race.

### Notification de-duplication

The extension retains the latest 50 notification push identifiers in
`chrome.storage.local`. A repeated tickle for a known identifier is skipped.
If Chrome fails to create a notification, the identifier is removed so a later
retry remains possible.

### Persistent diagnostic journal

The extension now retains the latest 100 auto-open batch records in
`chrome.storage.local` under `autoOpenDiagnosticLog`. Each record includes:

- event source, such as `websocket-tickle` or `refresh-pushes`;
- batch size;
- redacted push-ID suffixes for opened or already-opened pushes; and
- an ISO-8601 timestamp.

It deliberately excludes the Pushbullet access token, URLs, titles, and push
bodies.

## Verification

Static checks passed:

```text
git diff --check
node --check background.js
```

Manual evidence from the latest service-worker console shows exactly one
auto-open operation for the tested link, even after repeated WebSocket traffic.

After reloading the unpacked extension, inspect the retained diagnostic
records in its service-worker console:

```javascript
chrome.storage.local
  .get('autoOpenDiagnosticLog')
  .then(({ autoOpenDiagnosticLog }) => console.table(autoOpenDiagnosticLog));
```

For a new link push, expect one record that lists the push ID under
`openedPushIdens`. Later events for the same push may list it under
`alreadyOpenedPushIdens`, but must not open another tab.

## Local extension copy

Keep only the Git checkout loaded in Chrome:

```text
/Users/zyron/Documents/CalOne/github/pushbullet-chrome-extension
```

The older Downloads archive is an obsolete, separate unpacked-extension
candidate. Loading both copies can independently receive the same Pushbullet
event and reproduce duplicate behavior.
