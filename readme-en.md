# HL Service — Warehouse Slot Management System

> **A single-file, real-time warehouse localization system built to be as simple as possible to deploy and use — no framework, no build step, just open and run.**

[🇧🇷 Versão em Português](README_PT.md)

---

## Background

HL Service is an appliance repair shop. The warehouse stores thousands of spare parts — heating elements, compressors, control boards, thermostats, motors — sourced from dozens of brands and product lines.

The original storage logic was **category-based**: each physical box was reserved for a specific part type and brand combination. For example, one box held only *coffee maker heating elements from brand X*, another held *coffee maker heating elements from brand Y*, and so on. This seemed organized at first, but in practice it created serious problems:

- **Boxes locked to narrow categories sat nearly empty** while other categories overflowed, wasting physical space
- **New part types had no home** — operators would improvise, breaking the category logic silently
- **Brand proliferation created an explosion of micro-categories** — dozens of near-empty boxes that could have shared space
- **Nobody knew which box was full or available** — you had to walk to the shelf and open it to find out
- **Retrieval depended on memory** — if the person who put the part away wasn't around, finding it meant searching the whole floor
- **No audit trail** — there was no record of who moved what, or when, making inventory reconciliation impossible

---

## The Solution

A self-contained web application built in a **single HTML file**, connected to a Supabase PostgreSQL backend and optionally integrated with Tiny ERP for automatic location sync.

The core shift: instead of assigning boxes to categories, **every SKU gets a coordinate**. Any part can go in any box. The system tracks where it is. The category logic moves out of the physical layout and into the database.

Built to be as simple as possible to adopt in a real warehouse environment — no installation, no training on a complex UI, no dependency on any specific device. If it has a browser and can scan a barcode, it works.

The system introduces:

- **SKU-level slotting** — every item is assigned a precise coordinate: `Floor → Sector → Shelf → Box → Side`
- **Occupancy tracking** — each box/side has a four-level occupancy status (Empty / Low / Medium / Almost Full), making space utilization visible at a glance
- **Visual warehouse map** — a real-time overview panel renders every sector, shelf, and box with color-coded occupancy, so operators can identify available space without walking the floor
- **Barcode scanner support** — the search field triggers on Enter, compatible with any USB/Bluetooth barcode reader
- **ERP integration** — when a position is saved, the system automatically pushes the location to Tiny ERP via a local server bridge

---

## Features

### SKU Search & Registration
- Search any SKU by typing or scanning a barcode
- If the SKU exists, its current location is displayed and the operator can immediately update it
- If not found, a registration form opens pre-filled with the last used location (reducing repetitive input)
- Duplicate detection warns if a SKU is being registered twice, or if a box already exceeds its configured capacity

### Visual Overview Panel
- Renders the full warehouse in a hierarchical layout: sector → shelf → box grid
- Each box cell displays its name (C1, C2…) colored by its occupancy level:
  - 🟢 **Low** — green
  - 🟡 **Medium** — amber
  - 🔴 **Almost Full** — red
  - ⬛ **Empty** — dark gray
- Clicking a box expands a drawer showing every SKU inside, organized by side (A, B, C…)
- The **✏ Edit Occupancy** badge inside the drawer lets operators update the occupancy level per side without leaving the panel
- **Legend filters** — clicking any occupancy level in the legend dims and blurs all non-matching boxes, isolating what you want to see
- **SKU Match filter** — when a SKU search is active, clicking "Match SKU" in the legend highlights only the boxes containing that SKU; everything else is darkened
- **Scroll-to-zoom** — hovering over the box grid and scrolling zooms in/out for dense layouts

### Position Search
- Operators can look up all SKUs stored at a specific coordinate (floor + sector + shelf + box + side) without knowing the SKU code
- Useful for auditing and physical inventory reconciliation

### Movement History
- Every save action is recorded: SKU, operator name, previous position, new position, timestamp, and action type (registration / update / deletion)
- Accessible via the "History" button in the header

### Structure Management
- The warehouse layout (sectors, shelves, box counts, box capacity) is fully configurable from the UI via "Manage Structure"
- No hardcoded limits — adapts to any warehouse floor plan

### Tiny ERP Sync
- A local server (`INICIAR_SERVIDOR_ERP.bat`) bridges the web app to the Tiny ERP API
- When online, saving a position automatically updates the location field in Tiny
- When offline, positions are saved locally and can be bulk-synced later via "Sync ERP"

### Offline Resilience
- On connection loss, the system loads the last local backup and displays it with a warning
- All recent data is cached in localStorage as a fallback

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Vanilla HTML/CSS/JS — zero dependencies, zero build step |
| Database | Supabase (PostgreSQL) with Row-Level Security |
| ERP Bridge | Local Node.js server (Tiny ERP REST API) |
| Hosting | Any static file host, or opened locally in a browser |

---

## Database Schema

The core table (`estoque`) stores one row per SKU:

```sql
CREATE TABLE estoque (
  id             BIGSERIAL PRIMARY KEY,
  sku            TEXT NOT NULL UNIQUE,
  andar          TEXT,           -- floor (e.g. "2A", "3A")
  setor          TEXT,           -- sector (e.g. "6S")
  prateleira     TEXT,           -- shelf  (e.g. "P1")
  caixa          TEXT,           -- box    (e.g. "C3")
  lado           TEXT,           -- side   (e.g. "A", "B")
  nivel          TEXT DEFAULT 'pouca'
                 CHECK (nivel IN ('vazio','pouca','media','quase')),
  operador       TEXT,
  atualizado_em  TIMESTAMPTZ DEFAULT now(),
  sincronizado_erp_em TIMESTAMPTZ,
  erro_erp       TEXT
);
```

To add occupancy tracking to an existing installation:

```sql
ALTER TABLE estoque
  ADD COLUMN IF NOT EXISTS nivel TEXT NOT NULL DEFAULT 'pouca'
  CHECK (nivel IN ('vazio', 'pouca', 'media', 'quase'));

CREATE INDEX IF NOT EXISTS idx_estoque_nivel ON estoque (nivel);
```

---

## How to Run

1. Create a Supabase project and run the schema above
2. Open `HL_COORDENADAS.html` in any modern browser (Chrome recommended)
3. Fill in your Supabase URL and API key in the `CONFIG` section at the top of the `<script>` block
4. Optionally run `INICIAR_SERVIDOR_ERP.bat` on the warehouse PC for Tiny ERP sync
5. Use "Manage Structure" to define your floors, sectors, shelves, and box counts

No server, no npm install, no build pipeline — just open the file.

---

## Screenshots

> *(Insert screenshots here)*

| Overview Panel | Box Detail Drawer | SKU Search |
|---|---|---|
| ![overview](screenshots/overview.png) | ![drawer](screenshots/drawer.png) | ![search](screenshots/search.png) |

---

## Impact

| Before | After |
|---|---|
| Boxes restricted to narrow part categories (e.g. "brand X coffee maker heating elements only") | Any SKU can go in any box — space is filled by need, not by category |
| Brand proliferation created dozens of near-empty dedicated boxes | Boxes shared freely; occupancy visible in real time |
| Operators relied on memory or walked the floor to find items | Any operator locates any SKU in seconds via search or barcode scan |
| No feedback on box capacity | Color-coded occupancy across the entire warehouse map |
| Misplacement only discovered during picking | Duplicate and overflow warnings enforced at save time |
| No record of who moved what or when | Full movement history with operator name and timestamp |
| ERP location fields updated manually | Automatic ERP sync on every save |

---

## Author

Developed for **HL Service** internal logistics operations.
