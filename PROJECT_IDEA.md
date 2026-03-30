# Project Idea — Representantes de Casilla

## Concept

A mobile-first Progressive Web App (PWA) for capturing and managing **polling station representatives** (representantes de casilla) during Mexican elections. Field operators use their phones to register people assigned to cover specific voting booths on election day, syncing data in real-time to a central Google Sheets backend.

## Problem It Solves

Political parties need to coordinate hundreds of representatives across dozens of voting booths (casillas) within a municipality. Without a centralized system:
- Multiple capturers assign the same person or booth twice
- There's no visibility into which booths still need coverage
- Paper lists get lost, phones die, data is fragmented
- Coordinators can't see real-time progress

This app gives every field capturer a shared, offline-capable tool that prevents duplicates, tracks coverage, and syncs to a single source of truth.

## Target Users

1. **Field capturers** (capturistas) — People in the field registering representatives. They use the Capture and Records screens. They need fast data entry on mobile with minimal friction.
2. **Coordinators** — People monitoring coverage progress. They check the Dashboard and Consulta screens to see which booths are covered and which need attention.
3. **Administrators** — People who set up the system. They configure the municipality, server URL, catalog, and PIN in the Config screen.

## Core Workflow

1. **Setup**: Admin configures municipality name, Google Apps Script URL, downloads the booth catalog (sections + booths), sets a PIN, and lists the political parties.
2. **Capture**: Field capturer opens the app, enters PIN, and registers a representative by choosing:
   - **Casilla (booth-specific)**: Section → Booth → Role (Propietario/Suplente) → Personal data + INE photo
   - **General (multi-section)**: One or more sections → Personal data (no booth, no role, no photo)
3. **Sync**: Records auto-sync to Google Sheets via Apps Script POST. If offline, records queue locally and sync later.
4. **Monitor**: Dashboard shows totals, sync status, coverage per section, and general representatives list. Data comes from both local records and server state.
5. **Consult**: Anyone can look up a specific booth to see who's assigned to it (queries the server in real-time).
6. **Recover**: If local data is lost, all records can be recovered from the server.

## Two Types of Representatives

| Type | Casilla (Propietario/Suplente) | General |
|------|-------------------------------|---------|
| Scope | One specific booth in one section | One or more full sections |
| Role | Propietario (primary) or Suplente (alternate) | No role distinction |
| Max per booth | 1 propietario + 1 suplente | N/A |
| Max per section | N/A | 1 general per section |
| Photos | INE front + back (optional) | None |
| Color accent | Navy blue | Purple |

## Key Design Decisions

- **Single-file PWA**: Everything in one `index.html` — no build step, no dependencies, no framework. Deployed via GitHub Pages. This keeps deployment dead simple for non-technical coordinators.
- **Offline-first**: IndexedDB stores everything locally. The app works without internet. Sync happens when connectivity is available.
- **Google Sheets as backend**: The central database is a Google Sheet accessed via Apps Script. This was chosen because coordinators already know Sheets, can view/edit data there, and it requires zero server infrastructure.
- **no-cors POST**: Sync uses `mode: 'no-cors'` because Apps Script doesn't return CORS headers for POST. This means the app can't read the response — it assumes success if no network error.
- **PIN access**: Simple PIN-based auth (not user accounts). All capturers in a municipality share the same PIN. The PIN is stored locally, not on the server.
- **Duplicate prevention**: INE (voter ID) is checked against both local records and server state. Roles are blocked if already assigned in the server. Sections are checked for existing generals.
- **Accent normalization**: All names are stripped of accents and uppercased before storage to prevent encoding issues with Google Sheets.
- **Photo compression**: INE photos are resized to max 900px width and compressed to JPEG 65% quality to keep IndexedDB and sync payloads manageable.

## Data Flow

```
[Field Capturer Phone]          [Google Apps Script]          [Google Sheets]
        |                              |                            |
        |-- POST record (no-cors) ---->|---- appendRow() --------->|
        |                              |                            |
        |-- GET ?action=catalogo ----->|---- read Catalogo tab --->|
        |<-- { casillas: [...] } ------|                            |
        |                              |                            |
        |-- GET ?action=estado ------->|---- read Registros tab -->|
        |<-- { registros, generales }--|                            |
        |                              |                            |
        |-- GET ?action=consulta&... ->|---- filter by sec/cas --->|
        |<-- { representantes: [...] }-|                            |
        |                              |                            |
        |-- GET ?action=recuperar ---->|---- read all rows ------->|
        |<-- { registros, generales }--|                            |
```

## What Success Looks Like

- Every booth in the municipality has a propietario and suplente assigned before election day
- Every section has a general representative assigned
- Zero duplicate assignments (same person or same role/booth)
- Coordinators can see real-time coverage at a glance
- Data survives phone issues (offline storage + server recovery)
- Non-technical users can set it up and use it without training

## Scope Boundaries

This app does **not**:
- Handle user accounts or authentication beyond a shared PIN
- Store data on its own server (relies entirely on Google Sheets)
- Work across municipalities (one instance per municipality)
- Track election-day activities (it's a pre-election preparation tool)
- Generate reports or analytics (that's done in Google Sheets directly)
- Support editing or deleting individual records from the app (only bulk clear + recovery)
