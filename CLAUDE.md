# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Aviation Ground Speed Calculator — a bilingual (EN/UA) installable PWA that computes ground speed, heading, and wind correction angle from true airspeed, wind speed, and wind direction. It is a zero-dependency static site: no package manager, no bundler, no test runner, no lint config.

## Running locally

Because the app registers a service worker (`sw.js`), it must be served over HTTP — opening `index.html` directly via `file://` will not fully exercise the PWA. Any static server works, e.g.:

```
python3 -m http.server 8000
```

Then visit `http://localhost:8000/`. There are no build, lint, or test commands.

Deployment is just: publish the repository root as static files.

## Architecture

Everything runtime lives in three files at the repo root:

- **`index.html`** — the entire app. HTML markup, CSS (inline `<style>`), and JavaScript (inline `<script>`) are all in this single file. No external JS/CSS dependencies.
- **`sw.js`** — service worker using a cache-first strategy over a precache list (`ASSETS`). Falls back to `./index.html` when offline and the request misses the cache.
- **`manifest.json`** — PWA metadata and icon set (`icon.svg`, `icon-192.png`, `icon-512.png`, `icon-maskable-512.png`, `apple-touch-icon.png`).

### Calculation flow (`calculateGroundSpeed` in `index.html`)

Inputs: TAS (km/h), wind speed (m/s), course (deg), wind direction (deg, "from" convention). The function:

1. Converts TAS km/h → m/s; normalizes angles to `[0, 360)`.
2. Computes wind correction angle via `WCA = asin(WS·sin(WD − course) / TAS)`. If `|crosswind| > TAS`, surfaces `errorCrossWind` and aborts (this is the "calculation impossible" branch — preserve it).
3. Heading = normalize(course + WCA).
4. Ground speed via law of cosines: `GS = √(TAS² + WS² − 2·TAS·WS·cos(heading − WD))`, then m/s → km/h.

Outputs are written to the three readonly `<input class="output-field">` fields. `clearOutputs()` is used for any early return.

### i18n

A single `translations` object with `en` and `ua` keyed strings drives all UI text. `setLanguage(lang)` iterates translation keys and sets `textContent` on the element whose `id` matches the key. **To add a translatable string:** (1) add the key to *both* `en` and `ua`, and (2) give the corresponding DOM element an `id` equal to that key. Error/warning strings are looked up via `translations[currentLanguage]` inside `calculateGroundSpeed` rather than by DOM id, so they must exist in both languages too.

### Angle input normalization

`course` and `windDirection` inputs use `oninput="normalizeAngleInput(...)"` + `onblur="calculateGroundSpeed()"`. `normalizeAngleInput` rewrites out-of-range values in place, flashes a warning, and triggers recalculation. The other inputs recalc directly on `oninput`.

## Conventions to respect

- **Bump the service worker cache version when changing any cached asset.** `CACHE_NAME` in `sw.js` (currently `aviation-calc-v1.0.3`) must be incremented whenever `index.html`, `manifest.json`, icons, or the `ASSETS` list change — otherwise returning users will get stale content from the old cache. Also keep the `ASSETS` array in sync with the actual files shipped.
- **Keep the app a single HTML file.** Styles and scripts are inline by design; do not split them into separate `.css`/`.js` files without also updating `sw.js` ASSETS and the cache version.
- **No new runtime dependencies.** The app must work offline from the precache with zero network calls.
- **Preserve EN/UA parity.** Every user-facing string added in one language must be added in the other.
