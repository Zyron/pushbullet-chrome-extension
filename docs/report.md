# Release Report: Version 1.0.2
**URL Normalization & Service Worker API Optimizations**

This report documents the changes, bugs resolved, and optimizations implemented for the Pushbullet Chrome Extension in the **v1.0.2** update.

---

## 1. Executive Summary
The v1.0.2 update addresses critical bugs related to URL handling and network reliability. Specifically, it resolves:
1. **URL Scrambling (Relative Pathing)**: Link pushes without explicit protocol schemes resolving relative to the browser extension's internal origin (e.g., `comet-extension://`).
2. **API Nickname POST Spamming**: Excessive `POST` nickname updates hitting the Pushbullet API on every service worker activation.
3. **Verbose Offline Stack Traces**: Frequent `TypeError: Failed to fetch` console tracebacks when the browser is offline or waking up from sleep.

These fixes reduce network overhead, prevent Pushbullet API rate-limiting (`502 Bad Gateway` errors), and optimize memory cache longevity.

---

## 2. Detailed Problem Analysis

### Issue A: URL Scrambling (Relative Tab Resolution)
When `chrome.tabs.create({ url })` or an HTML `<a>` tag's `href` attribute receives a web link that lacks a protocol scheme (for example, `claude.com` or `claude.com/blog/...` instead of `https://claude.com`), the browser treats it as a relative URL. 
* **Outcome**: The browser resolves it relative to the extension's origin, yielding paths like `chrome-extension://[id]/claude.com` or `comet-extension://jfdiicjllkdabglanennlphpdnaoeohk/claude.com`.

### Issue B: Ephemeral Service Worker Nickname POST Spam
Under Manifest V3, background service workers are ephemeral and stop/restart frequently (after 30 seconds of inactivity). 
* **Mechanism**: On every service worker startup, `initializeSessionCache()` was running `registerDevice()`, which in turn invoked `updateDeviceNickname()`—making a `POST` request to `https://api.pushbullet.com/v2/devices/[iden]`.
* **Outcome**: The extension flooded the Pushbullet API with identical nickname update requests multiple times per minute, resulting in server-side `502 Bad Gateway` failures and rate-limiting.

### Issue C: Ephemeral Service Worker Cache Staleness
`sessionCache` (user info, devices, and pushes) was stored in memory variables inside the service worker. 
* **Mechanism**: Every time the service worker restarted (e.g., waking up for keep-alive alarms, WebSocket messages, or popup clicks), memory was cleared, prompting a clean cache initialization and triggering three API calls (`fetchUserInfo`, `fetchDevices`, `fetchRecentPushes`).

---

## 3. Technical Solutions Implemented

### 1. URL Normalization
We implemented a robust `normalizeUrl(url)` helper function inside both `background.js` and `js/popup.js`:
```javascript
function normalizeUrl(url) {
  if (!url) return '';
  const trimmed = url.trim();
  
  // Check if URL has a valid scheme or is protocol-relative
  const hasScheme = /^[a-zA-Z][a-zA-Z0-9+.-]*:\/\//.test(trimmed) || /^(mailto|tel|sms|magnet):/i.test(trimmed);
  if (hasScheme) return trimmed;
  if (trimmed.startsWith('//')) return 'https:' + trimmed;
  
  return 'https://' + trimmed;
}
```
This is applied dynamically when:
* Auto-opening pushes via the WebSocket in the background.
* Clicking notification banners.
* Rendering lists in the popup interface.
* Pushing new links manually from the popup form (ensuring correct format before it reaches Pushbullet's servers).

### 2. Startup Nickname Sync Safeguard
We introduced a tracking flag `lastSyncedDeviceNickname` in `chrome.storage.local`. During startup:
* The extension checks if `lastSyncedDeviceNickname` matches the current `deviceNickname`.
* The `POST` nickname update request is bypassed unless the nickname has changed.
* On successful sync, `lastSyncedDeviceNickname` is updated.

### 3. Persistent Local Caching
To maintain cached lists across service worker terminations:
* The `sessionCache` object is written to `chrome.storage.local` upon updates.
* Upon service worker reactivation, `sessionCache` is restored immediately from local storage.
* Network fetch calls are skipped unless the cache is stale (older than 30s) and the browser is online.

### 4. Connection State Awareness & Error Logging
* Added checks for `navigator.onLine` before executing network-bound actions (WebSocket connections, keep-alive alarms, cache refreshes).
* Modified error blocks to intercept routine `Failed to fetch` errors when offline, logging them as clean warning messages instead of full stack trace failures.

---

## 4. File-by-File Change Log

### [manifest.json](file:///Users/zyron/Documents/CalOne/github/pushbullet-chrome-extension/manifest.json)
* Bumped version number to `1.0.2`.

### [background.js](file:///Users/zyron/Documents/CalOne/github/pushbullet-chrome-extension/background.js)
* Added `normalizeUrl` utility.
* Integrated `normalizeUrl` in `pushLink`, `openPushLink`, and notification clicks.
* Added `navigator.onLine` checks in `connectWebSocket`, keep-alive alarm listeners, `refreshPushes`, `refreshSessionCache`, and `updateDeviceNickname`.
* Implemented reading/writing `sessionCache` and `lastSyncedDeviceNickname` using `chrome.storage.local`.
* Optimized startup check in `initializeSessionCache` to avoid fetching from the API if cached data is fresh.
* Sanitized network errors to display as warnings when offline.
* Added cleanup of cache storage keys when clearing API keys on logout.

### [js/popup.js](file:///Users/zyron/Documents/CalOne/github/pushbullet-chrome-extension/js/popup.js)
* Added `normalizeUrl` utility.
* Integrated `normalizeUrl` in popup UI render (`displayPushes`) and manual push inputs (`sendPush`).
* Updated `logout` to remove `sessionCache` and `lastSyncedDeviceNickname` from storage.
