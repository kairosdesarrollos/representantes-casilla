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

## Project Idea

Read `PROJECT_IDEA.md` for the full concept — what the app does, who it's for, the core workflow, design decisions, data flow, and scope boundaries. Any shift in the product concept, target users, workflow, or scope must be reflected in `PROJECT_IDEA.md`.

## Technical Reference

Read `TECHNICAL_REFERENCE.md` for a complete technical map of the codebase — global state, IndexedDB schema, all functions with line numbers, HTML element IDs, CSS class conventions, backend endpoints, validation rules, and capture flow details.

**Important**: Any change to the technical architecture (new functions, renamed IDs, new screens, schema changes, new endpoints, CSS variable changes, etc.) must be reflected in `TECHNICAL_REFERENCE.md` to keep it in sync with the code.
