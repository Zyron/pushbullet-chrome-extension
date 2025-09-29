# Repository Guidelines

## Project Structure & Module Organization
The extension is intentionally small. `background.js` hosts the service worker that authenticates with Pushbullet, keeps the WebSocket alive, and manages context menus. The popup UI lives in `popup.html`, styled by `css/popup.css` and driven by `js/popup.js`. Icons for store packaging sit under `icons/`. `manifest.json` defines permissions, background/service worker wiring, and context menus—review it first when changing capabilities.

## Build, Test, and Development Commands
- Load for development via Chrome: `chrome://extensions` → enable Developer Mode → `Load unpacked` pointing to the repo root. Reload here after each edit.
- Package for distribution with `zip -r pushbullet-chrome.zip background.js css icons js manifest.json popup.html` (run from the repo root). Upload the archive to the Chrome Web Store dashboard when releasing.

## Coding Style & Naming Conventions
Use two-space indentation across JavaScript, HTML, and CSS. Prefer `const` and `let` over `var`, keep imports at the top (when added), and favor `async/await` for API calls as in `background.js`. Strings should use single quotes; template literals are reserved for interpolation. Name functions and variables descriptively in lower camelCase (`connectWebSocket`, `autoOpenLinks`), and keep inline comments concise and purposeful.

## Testing Guidelines
There is no automated test suite yet; rely on manual verification in Chrome. After changes, reload the unpacked extension and confirm: authentication persists between popup opens, link pushes auto-open when the service worker restarts, context menu items appear for pages, selections, and images, and the popup displays recent pushes without console errors (`chrome://extensions` → "background page" logs). When adding tests, mirror the feature name in filenames (e.g., `autoOpenLinks.test.js`) and colocate them next to the code they exercise.

## Commit & Pull Request Guidelines
Follow the existing Git history: short, imperative commit subjects (`Improve auto-opening of received links`). Group related changes together and avoid mixed concerns. Pull requests should outline the intent, list testing performed, and link any related Pushbullet issues or forum threads. Include screenshots or screen recordings for UI adjustments so reviewers can see popup or notification outcomes.

## Security & Configuration Tips
Never commit personal Pushbullet access tokens or exported user data. Sanitize console logging before release builds, and confirm new permissions or host matches are justified in `manifest.json` to avoid Chrome review delays.
