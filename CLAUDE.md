# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Sistema de captura de representantes de casilla (polling station representatives capture system). This is a **single-file PWA** (`index.html`) designed for mobile use, written entirely in Spanish.

## Architecture

The entire app lives in `index.html` — HTML, CSS, and JS are all inline. There is no build system, bundler, or package manager.

### Key Patterns

- **State**: Global `S` object holds all app state (config, catalogo, current form data, screens)
- **Storage**: IndexedDB (`RepCasillaDB`) with stores: `config`, `catalogo`, `registros`, `estado`
- **Backend**: Google Apps Script deployed as web app — the `scriptUrl` in config points to it. All sync operations use `fetch()` against this URL with `?action=` query params (`catalogo`, `estado`, `recuperar`) or POST for saving
- **Navigation**: Tab-based screens (`#scr-dash`, `#scr-cap`, `#scr-reg`, `#scr-cfg`) controlled by `showScreen()`
- **Two capture flows**: "Casilla" (per-booth with section/booth/role selection) and "General" (multi-section, different steps)
- **Photos**: Captured via `<input type="file" capture="environment">`, compressed to JPEG with canvas, stored as base64 data URLs in IndexedDB

### CSS Variables

Design system uses CSS custom properties on `:root` — navy/red/green/amber/purple color scheme with `--fh` (Barlow Condensed), `--fb` (Barlow), `--fm` (JetBrains Mono) font families.

## Development

Open `index.html` directly in a browser or serve with any static server:

```
npx serve .
# or
python3 -m http.server
```

No tests, no linter, no build step. Changes are made directly to `index.html`.

## Deployment

Hosted via GitHub Pages from the `main` branch.
