# Technical Reference — Representantes de Casilla PWA

> Complete technical map of `index.html` (~2005 lines). Last updated: 2026-03-30.

---

## File Structure (line ranges)

| Section | Lines | Description |
|---------|-------|-------------|
| HTML head + CSS | 1–349 | Meta tags, Google Fonts, full CSS design system |
| Login screen | 352–362 | PIN-based auth overlay |
| App shell | 364–404 | Header, sync button, nav tabs (5 screens) |
| Dashboard screen | 407–444 | Stats, coverage list, generals list |
| Capture screen | 447–778 | Type selector + Casilla form (4 steps) + General form (3 steps) |
| Records screen | 782–791 | Filter bar + record list |
| Config screen | 794–863 | Settings cards (municipality, sync URL, catalog, PIN, parties, data mgmt) |
| Consulta screen | 866–894 | Remote booth lookup |
| Toast + Overlay | 897–898 | Notification and loading overlay |
| JS: State (`S`) | 905–916 | Global state object |
| JS: IndexedDB | 921–938 | DB open, helpers (`tx`, `dbAll`, `dbPut`, `dbClear`) |
| JS: Config | 943–981 | `loadCfg()`, `saveCfg()`, `refreshHdr()`, `parsePars()` |
| JS: Catalog | 986–1007 | `loadCatalogo()`, `descargarCatalogo()`, `updateCatStatus()` |
| JS: Flow control | 1012–1022 | `iniciarFlujo()`, `volverTipo()` |
| JS: Casilla flow | 1027–1115 | Steps 1-4, validation, save, reset |
| JS: General flow | 1120–1226 | Steps 1-3, multi-section, save, reset |
| JS: Section search | 1231–1276 | Shared search for both flows + consulta |
| JS: Global state (Sheets) | 1281–1360 | `descargarEstado()`, role blocking, banner |
| JS: Casilla grid | 1365–1421 | Grid rendering with local+global status |
| JS: Party/Role selection | 1426–1436 | `renderCPar()`, `renderGPar()`, `selRol()` |
| JS: Photos | 1441–1461 | Camera/gallery capture, canvas compression |
| JS: Validations | 1466–1566 | Email, INE (regex), phone, name normalization |
| JS: Records | 1571–1592 | `loadReg()`, `renderReg()`, `filt()` |
| JS: Dashboard | 1597–1695 | `updateDash()` — stats, coverage bars, generals list |
| JS: Sync | 1703–1788 | `syncOne()`, `syncNow()`, `testConn()` |
| JS: Consulta | 1793–1898 | Remote booth query via Apps Script |
| JS: Navigation | 1903–1920 | `showScreen()` |
| JS: Utilities | 1925–1957 | `toast()`, `showOv()`, `hideOv()`, `exportJSON()`, `clearAll()`, `recuperarDeSheets()` |
| JS: Init | 1962–2002 | `init()` — DB open, config load, event listeners, auto-catalog |

---

## Global State Object (`S`)

```js
const S = {
  // Casilla capture state
  cStep: 1,              // Current step (1-4)
  cSeccion: null,        // { seccion: string }
  cCasilla: null,        // { id_casilla, seccion, tipo_casilla }
  cRol: null,            // 'PROPIETARIO' | 'SUPLENTE'
  cPartido: null,        // Party name string
  fotoFrente: null,      // base64 data URL
  fotoReverso: null,     // base64 data URL

  // General capture state
  gStep: 1,              // Current step (1-3)
  gSecciones: [],        // Array of section strings
  gPartido: null,        // Party name string

  // Data
  catalogo: [],          // Booth catalog from IndexedDB
  registros: [],         // All local records
  filtro: 'all',         // Records filter

  // Remote state from Google Sheets
  estadoGlobal: [],      // Casilla reps from server
  generalesGlobal: [],   // General reps from server
  estadoTs: null,        // Last fetch timestamp

  // Config
  cfg: {
    municipio: '',       // Municipality name
    capturista: '',      // Capturer email
    scriptUrl: '',       // Google Apps Script URL
    partidos: 'PAN',     // Comma-separated party list
    pin: '',             // Access PIN (4-8 digits)
    inact: '30',         // Inactivity timeout minutes
    deviceId: ''         // Auto-generated device ID
  }
};
```

---

## IndexedDB Schema

**Database**: `rep_casillas_v4` (version 1)

| Store | Key | Fields |
|-------|-----|--------|
| `registros` | `id` (autoIncrement) | tipo_reg, seccion, id_casilla, tipo_casilla, nombre, nombre_p, ap_pat, ap_mat, ine, telefono, rol, partido, secciones (general only), notas, foto_frente, foto_reverso, capturado_por, device_id, timestamp, synced, sync_error |
| `config` | `k` (string) | k, v — key-value pairs for all cfg fields |
| `catalogo` | `id` (string: `${seccion}-${id_casilla}`) | id, seccion, id_casilla, tipo_casilla |

Helper functions: `tx(store, mode)`, `dbAll(store)`, `dbPut(store, obj)`, `dbClear(store)`

---

## Screens & Navigation

5 tab screens controlled by `showScreen(name)`:

| Screen ID | Tab Label | Key Behavior |
|-----------|-----------|--------------|
| `screen-dashboard` | Inicio | Stats, coverage bars, generals list, sync button |
| `screen-captura` | Captura | Type selector → Casilla (4 steps) or General (3 steps) |
| `screen-registros` | Registros | Filtered list of all records with status badges |
| `screen-consulta` | Consulta | Remote booth lookup against Google Sheets |
| `screen-config` | Config | Municipality, sync URL, catalog, PIN, parties, data management |

---

## Capture Flows

### Casilla Flow (Propietario/Suplente) — 4 Steps

1. **Section selection** — Search section from catalog, display section info
2. **Booth selection** — Grid of booths with status indicators (libre/local/parcial/completa from Sheets)
3. **Data entry** — Role (PROPIETARIO/SUPLENTE with blocking), Party, Name, INE (18-char validated), Phone (10 digits), Photos (front/back with canvas compression to 900px JPEG 65%), Notes
4. **Confirmation** — Review all data → `guardarCasilla()`

### General Flow (Representante General) — 3 Steps

1. **Section selection** — Multi-select sections with tags
2. **Data entry** — Party, Name, INE, Phone, Notes (no photos, no role)
3. **Confirmation** — Review → `guardarGeneral()`

---

## Validation Rules

| Field | Rule | Regex/Logic |
|-------|------|-------------|
| INE (Clave Elector) | 18 chars: 6 letters + 8 digits + H/M + 3 alphanumeric | `/^[A-Z]{6}[0-9]{8}[HM][A-Z0-9]{3}$/` |
| Phone | Exactly 10 digits | Strip non-digits, check length |
| Name/Apellido | Min 2 chars, auto-uppercase, accents stripped | `quitarAcentos()` via NFD normalization |
| Email (capturer) | Standard email pattern | `/^[^\s@]+@[^\s@]+\.[^\s@]+$/` |
| INE duplicate | Checked against local `S.registros` + `S.estadoGlobal` + `S.generalesGlobal` | |
| Section uniqueness (General) | One general per section, checked local + Sheets | |
| Role blocking (Casilla) | Roles occupied in Sheets are visually blocked | `actualizarRolesGlobal()` |

---

## Backend Integration (Google Apps Script)

All communication via `fetch()` to `S.cfg.scriptUrl`.

### GET Endpoints (query params)

| Action | URL | Response |
|--------|-----|----------|
| `?test=1` | Connection test | `{ status: 'OK' }` |
| `?action=catalogo` | Download booth catalog | `{ casillas: [{ seccion, id_casilla, tipo_casilla }] }` |
| `?action=estado` | Get all reps from server | `{ status: 'OK', registros: [...], generales: [...] }` |
| `?action=recuperar` | Recover all records | `{ status: 'OK', registros: [...], generales: [...] }` |
| `?action=consulta&seccion=X&casilla=Y` | Query specific booth | `{ status: 'OK', representantes: [{ nombre, rol, partido, telefono }] }` |

### POST (save record)

- Method: POST, mode: `no-cors`
- Body: JSON of the full record object
- No response parsed (no-cors opaque response) — success assumed if no network error

---

## Sync Strategy

- **`syncOne(reg)`**: Called immediately after save. Sends single record via POST no-cors. Marks `synced: true` on success, `sync_error` on failure.
- **`syncNow()`**: Batch sync of all pending records (not synced, no error). Shows overlay. Sequential POST per record.
- **Auto-sync**: After each save, `syncOne()` is called automatically.
- **Offline**: Records saved locally in IndexedDB. Sync retried via manual `syncNow()` or header sync button.
- **Badge**: Sync button shows pending count badge in header.

---

## Photo Handling

- Input: `<input type="file" capture="environment">` (camera) or `<input type="file">` (gallery)
- Processing: `FileReader` → `Image` → Canvas resize (max 900px width) → `toDataURL('image/jpeg', 0.65)`
- Storage: base64 data URLs in `foto_frente` / `foto_reverso` fields of the record in IndexedDB
- Only available in Casilla flow (not General)

---

## Authentication & Security

- **PIN**: 4-8 digit PIN stored in IndexedDB config. Default `'1234'` if not set.
- **Login screen**: Overlay (`#login-screen`) shown on load if PIN configured. Hidden if no PIN.
- **Inactivity**: Timer resets on click/touch/keydown/scroll. Configurable: 15/30/60/120 min or never. Locks to login screen on timeout.
- **Device ID**: Auto-generated persistent ID (`DEV-{timestamp36}-{random}`), stored in config.

---

## CSS Design System

### Colors (CSS custom properties)

| Variable | Value | Usage |
|----------|-------|-------|
| `--navy` | `#1a2744` | Primary brand, headers, buttons |
| `--navy-d` | `#0f1829` | Nav background |
| `--red` | `#c8102e` | Errors, danger, active tab accent |
| `--green` | `#1a7a4a` | Success, synced states |
| `--amber` | `#d97706` | Warnings, pending states |
| `--purple` | `#6d28d9` | General representative flow accent |
| `--bg` | `#eef1f8` | Page background |
| `--surface` | `#fff` | Card backgrounds |
| `--border` | `#e2e8f0` | Borders |
| `--text` | `#1a2033` | Primary text |
| `--muted` | `#64748b` | Secondary text |
| `--light` | `#94a3b8` | Tertiary/hint text |
| `--r` | `12px` | Large border radius |
| `--rs` | `8px` | Small border radius |

### Fonts

| Variable | Family | Usage |
|----------|--------|-------|
| `--fh` | Barlow Condensed | Headings, badges, numbers |
| `--fb` | Barlow | Body text, buttons |
| `--fm` | JetBrains Mono | INE codes, phone numbers, device IDs |

---

## Key Functions Index

### Init & Config
- `init()` — App bootstrap (line 1962)
- `openDB()` — IndexedDB setup (line 922)
- `loadCfg()` / `saveCfg()` — Config load/save (line 943/961)
- `getDeviceId()` — Persistent device ID (line 1477)

### Capture
- `iniciarFlujo(tipo)` — Start casilla or general flow (line 1012)
- `volverTipo()` — Return to type selector (line 1017)
- `casGoStep(n)` — Navigate casilla steps (line 1028)
- `genGoStep(n)` — Navigate general steps (line 1121)
- `guardarCasilla()` — Save casilla record (line 1073)
- `guardarGeneral()` — Save general record (line 1168)
- `resetCasilla()` / `resetGeneral()` — Reset form state (line 1099/1211)

### Search & Selection
- `searchSec(q, prefijo)` — Shared section search for c/g/q prefixes (line 1231)
- `selCSec(sec)` — Select section in casilla flow (line 1267)
- `selCCas(id)` — Select booth in casilla flow (line 1398)
- `renderCasGrid()` — Render booth grid with status (line 1365)
- `addGenSec(sec)` / `removeGenSec(sec)` — Multi-section for general flow (line 1132/1140)

### Remote State
- `descargarEstado()` — Fetch global state from Sheets (line 1281)
- `descargarCatalogo()` — Fetch booth catalog (line 987)
- `getEstadoCasilla(seccion, id_casilla)` — Filter global state for booth (line 1323)
- `actualizarRolesGlobal()` — Block occupied roles (line 1331)
- `consultarCasilla()` — Remote booth query (line 1821)
- `recuperarDeSheets()` — Full data recovery from server (line 1931)

### Sync
- `syncOne(reg)` — Async single-record sync (line 1704)
- `syncNow(silent)` — Batch sync all pending (line 1736)
- `testConn()` — Test server connection (line 1772)

### UI
- `showScreen(name)` — Tab navigation (line 1903)
- `updateDash()` — Refresh dashboard stats and lists (line 1597)
- `renderReg()` — Render records list with filter (line 1572)
- `toast(msg, type, dur)` — Notification toast (line 1925)
- `showOv(msg)` / `hideOv()` — Loading overlay (line 1926/1927)

### Validation
- `valINE(input, cntId, erId)` — INE 18-char + regex validation (line 1549)
- `valTel(input, fmtId, erId)` — Phone 10-digit validation (line 1566)
- `valNom(input)` — Name validation + normalize (line 1547)
- `valAp(input, erId)` — Last name validation + normalize (line 1548)
- `valEmail(input)` — Email validation (line 1466)
- `quitarAcentos(str)` — NFD accent removal + uppercase (line 1536)

---

## HTML Element IDs (Key)

### Screens
`login-screen`, `app-screen`, `screen-dashboard`, `screen-captura`, `screen-registros`, `screen-consulta`, `screen-config`

### Dashboard
`dash-muni`, `dash-total`, `dash-total-gen`, `dash-global-lbl`, `st-sync`, `st-pend`, `st-err`, `big-sync`, `big-sync-lbl`, `sync-badge`, `cov-list`, `gen-dash-lbl`, `gen-dash-list`

### Capture — Casilla
`tipo-selector`, `form-casilla`, `cas-step-lbl`, `cas-sdots`, `cs-1` to `cs-4`, `c-sec-search`, `c-sec-res`, `c-sec-info`, `c-cas-grid`, `c-cas-info`, `c-par-grid`, `rol-prop`, `rol-sup`, `bloq-prop`, `bloq-sup`, `c-nom`, `c-pat`, `c-mat`, `c-ine`, `c-tel`, `c-not`, `foto-box-f`, `foto-box-r`, `foto-prev-f`, `foto-prev-r`, `cv-sec`, `cv-cas`, `cv-tip`, `cv-rol`, `cv-nom`, `cv-ine`, `cv-tel`, `cv-par`, `cv-not`, `cc-fotos`

### Capture — General
`form-general`, `gen-step-lbl`, `gen-sdots`, `gs-1` to `gs-3`, `g-sec-search`, `g-sec-res`, `g-sec-tags`, `g-sec-count`, `g-par-grid`, `g-nom`, `g-pat`, `g-mat`, `g-ine`, `g-tel`, `g-not`, `gv-secs`, `gv-par`, `gv-nom`, `gv-ine`, `gv-tel`, `gv-not`

### Config
`cfg-muni`, `cfg-cap`, `cfg-url`, `cfg-par`, `cfg-pin`, `cfg-inact`, `cfg-device-id`, `cat-status-wrap`

### Consulta
`q-sec-search`, `q-sec-res`, `q-cas-fld`, `q-cas-grid`, `q-btn-consultar`, `q-result`

---

## CSS Class Naming Conventions

- **Screens**: `.screen`, `.screen.active`
- **Cards**: `.cc` (confirm card), `.cfg-card`, `.stat-card`, `.cov-item`, `.reg-item`, `.gen-item`, `.result-card`
- **Status badges**: `.bp` (pending/amber), `.bs` (synced/green), `.be` (error/red), `.bgen` (general/purple)
- **Booth chips**: `.cas-chip`, `.cas-chip.sel`, `.cas-chip.has-local`, `.cas-chip.g-partial`, `.cas-chip.g-full`
- **Buttons**: `.btn`, `.btn-primary`, `.btn-secondary`, `.btn-success`, `.btn-danger`
- **Form fields**: `.fld`, `.fld input.ok`, `.fld input.bad`, `.errmsg`, `.errmsg.on`
- **Flow-specific accent**: `.gen` modifier turns elements purple (headers, badges, buttons, dots)
