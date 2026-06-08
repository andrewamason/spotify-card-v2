# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

A Home Assistant Lovelace custom card that displays a user's Spotify playlists and available playback devices. It is a fork of the original `spotify-card` maintained to stay compatible with [spotcast](https://github.com/fondberg/spotcast) v4+. The card is distributed as a single bundled JS file (`dist/spotify-card-v2.js`) installed into Home Assistant's `www` directory.

**Runtime dependencies**: The [Home Assistant Spotify Integration](https://www.home-assistant.io/integrations/spotify/) and the [spotcast custom component](https://github.com/fondberg/spotcast) must both be installed in Home Assistant for the card to function.

## Commands

```bash
npm install          # install dependencies
npm run build        # lint then bundle to dist/
npm run start        # watch mode (no lint, no minification)
npm run lint         # eslint with auto-fix
npm test             # jest unit tests
npm run coverage     # jest with coverage report
npm run rollup       # bundle only (skips lint)
npm run release      # lint + test + bundle (use before tagging a release)
```

The build output is `dist/spotify-card-v2.js` (plus a `.js.map` sourcemap). The `dist/` folder is committed and is what HACS/users install.

## Architecture

### Source files (`src/`)

| File | Role |
|---|---|
| `spotify-card-v2.ts` | Main `SpotifyCard` LitElement — renders the card, owns all reactive state, handles all user interaction |
| `spotcast-connector.ts` | All Home Assistant WebSocket calls via the `spotcast` custom component; owns device/player/playlist fetching logic |
| `editor.ts` | `SpotifyCardEditor` LitElement — the visual config editor shown inside Lovelace's card picker |
| `types.ts` | All shared TypeScript interfaces/enums (`SpotifyCardConfig`, `Playlist`, `ConnectDevice`, `CurrentPlayer`, etc.) |
| `const.ts` | Single `CARD_VERSION` export — bump this on every release |
| `localize/localize.ts` | Reads `localStorage.selectedLanguage` (or `navigator.language`) and returns translated strings from JSON files in `localize/languages/` |

### Data flow

1. `SpotifyCard.connectedCallback` subscribes to HA entity updates via `home-assistant-js-websocket` → `entitiesUpdated()` fires on every media-player state change.
2. State changes debounce into `updateSpotcast()` (500 ms), which calls `SpotcastConnector.updateState()` then `fetchPlaylists()`.
3. `SpotcastConnector` issues WebSocket messages of type `spotcast/devices`, `spotcast/player`, `spotcast/castdevices`, and `spotcast/playlists` to the spotcast HA integration. Results are written directly onto the parent `SpotifyCard`'s reactive `@property` fields (`devices`, `player`, `chromecast_devices`, `playlists`).
4. LitElement's reactive properties trigger re-renders. Custom `hasChanged` guards (`hasChangedCustom`, `hasChangedMediaPlayer`) avoid unnecessary renders for arrays and media-player state.
5. Playback is initiated by calling `hass.callService('spotcast', 'start', ...)` with either `device_name` (Chromecast) or `spotify_device_id` (Spotify Connect).

### Device priority and types

The card knows about three kinds of playback device:
- **Spotify Connect devices** — discovered via `spotcast/devices` WS call; normalized from `IncomingConnectDevice` → `ConnectDevice` to handle API differences between spotcast versions.
- **Known Connect devices** — user-configured in `SpotifyCardConfig.known_connect_devices`; useful for Sonos and other speakers not auto-discovered.
- **Chromecast devices** — discovered via `spotcast/castdevices`.

When a device is selected and no playback is active the connector falls back to playing the first playlist. When playback is already active, selecting a device transfers it.

### Dual state tracking for volume/playback controls

The card tracks two optional HA entity states:
- `_spotify_state` — the `media_player.spotify_*` entity (the Spotify integration).
- `_connect_player_state` — a HA entity for the currently active known Connect device (e.g. a Sonos entity), populated only when the active device matches a `known_connect_devices` entry that has an `entity_id`.

Playback controls and volume prefer `_connect_player_state` when present, falling back to `_spotify_state`. `shouldUpdate` suppresses re-renders for 500 ms after a volume change to prevent slider jumping.

### Editor

`SpotifyCardEditor` is registered as `spotify-card-v2-editor`. On connect it fetches `spotcast/accounts` and `spotcast/castdevices` to populate dropdowns. Configuration is split into three collapsible sections (General / Appearance / Advanced). All changes fire a `config-changed` event consumed by Lovelace.

The `PLAYLIST_TYPES` export from `editor.ts` must be imported by `spotify-card-v2.ts` at startup (there is an intentional workaround for a hard-to-debug Rollup/LitElement initialization order bug — do not remove that import).

## Build System

Rollup bundles `src/spotify-card-v2.ts` (which imports `editor.ts`) into a single ES module. In watch mode (`npm start`) Terser minification is skipped. The TypeScript compiler is configured with `noEmit: true` — Rollup handles transpilation via `rollup-plugin-typescript2`.

## Code Style

- Prettier: `printWidth: 120`, single quotes, trailing commas (ES5 style).
- ESLint extends `airbnb-base` + `@typescript-eslint/recommended`; `@typescript-eslint/no-explicit-any` is disabled.
- `noImplicitAny` is disabled in `tsconfig.json` despite `strict: true`.
- Unused parameters must be prefixed with `_` to suppress the ESLint warning.

## Localization

Add new user-facing strings to all three language files under `src/localize/languages/` (`en.json`, `de.json`, `se.json`). Reference them with `localize('section.key')`.

## Releasing

1. Bump `CARD_VERSION` in `src/const.ts`.
2. Run `npm run release` to lint, test, and rebuild `dist/`.
3. Commit the updated `dist/spotify-card-v2.js` along with the source changes.
4. Tag the commit — GitHub Actions (`release.yml`) handles the release creation.
