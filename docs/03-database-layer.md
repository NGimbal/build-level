# Module 03: Database Layer

## Overview

D1 database schema design, migrations, seed data, and type-safe query helpers for the backend.

## Goals

- Design normalized database schema for D1 (SQLite)
- Create migration scripts
- Seed default cost code format
- Implement type-safe query helpers
- Handle database access patterns efficiently

## File Structure

```
packages/backend/src/db/
├── schema.sql           # Database schema DDL
├── seed.sql             # Default data (cost codes)
├── queries.ts           # Type-safe query helpers
├── types.ts             # Database row types
└── migrations/          # Future migrations (optional)
    └── 001_initial.sql
```

## Database Schema

### Entity Relationship Diagram

```
┌─────────────────────┐       ┌─────────────────────┐
│  cost_code_formats  │       │     cost_codes      │
├─────────────────────┤       ├─────────────────────┤
│ id (PK)             │──┐    │ id (PK)             │
│ name                │  │    │ format_id (FK)      │──┐
│ version             │  └───▶│ code                │  │
│ description         │       │ label               │  │
│ created_at          │       │ level               │  │
│ updated_at          │       │ parent_code         │  │
└─────────────────────┘       │ sort_order          │  │
                              │ keywords (JSON)     │  │
                              │ description         │  │
                              └─────────────────────┘  │
                                                       │
┌─────────────────────┐       ┌─────────────────────┐  │
│      sessions       │       │     estimates       │  │
├─────────────────────┤       ├─────────────────────┤  │
│ id (PK)             │──┐    │ id (PK)             │  │
│ format_id (FK)      │──│───▶│ session_id (FK)     │──┘
│ status              │  │    │ filename            │
│ progress_total      │  │    │ r2_key              │
│ progress_completed  │  │    │ contractor_*        │
│ progress_current    │  │    │ client_*            │
│ error_message       │  │    │ subtotal            │
│ created_at          │  │    │ tax                 │
│ updated_at          │  │    │ grand_total         │
└─────────────────────┘  │    │ status              │
                         │    │ error_message       │
                         │    │ raw_text            │
                         │    │ created_at          │
                         │    │ updated_at          │
                         │    └─────────────────────┘
                         │              │
                         │              │
                         │    ┌─────────▼───────────┐
                         │    │     line_items      │
                         │    ├─────────────────────┤
                         │    │ id (PK)             │
                         │    │ estimate_id (FK)    │
                         │    │ description         │
                         │    │ section_header      │
                         │    │ quantity            │
                         │    │ unit                │
                         │    │ unit_price          │
                         │    │ total_price         │
                         │    │ line_number         │
                         │    │ cost_code           │
                         │    │ confidence          │
                         │    │ is_manual_override  │
                         │    │ alternate_codes     │
                         │    │ created_at          │
                         │    │ updated_at          │
                         │    └─────────────────────┘
                         │              │
                         │    ┌─────────▼───────────┐
                         │    │   estimate_notes    │
                         │    ├─────────────────────┤
                         │    │ id (PK)             │
                         │    │ estimate_id (FK)    │
                         │    │ note_type           │
                         │    │ content             │
                         │    └─────────────────────┘
                         │
                         │    ┌─────────────────────┐
                         │    │    comparisons      │
                         └───▶├─────────────────────┤
                              │ id (PK)             │
                              │ session_id (FK, UQ) │
                              │ comparison_data     │
                              │ generated_at        │
                              └─────────────────────┘
```

### Schema DDL (schema.sql)

```sql
-- Cost code formats (classification systems)
CREATE TABLE IF NOT EXISTS cost_code_formats (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  version TEXT NOT NULL,
  description TEXT,
  created_at TEXT DEFAULT (datetime('now')),
  updated_at TEXT DEFAULT (datetime('now'))
);

-- Individual cost codes within a format
CREATE TABLE IF NOT EXISTS cost_codes (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  format_id TEXT NOT NULL,
  code TEXT NOT NULL,
  label TEXT NOT NULL,
  level INTEGER NOT NULL,
  parent_code TEXT,
  sort_order INTEGER NOT NULL DEFAULT 0,
  keywords TEXT,  -- JSON array
  description TEXT,
  FOREIGN KEY (format_id) REFERENCES cost_code_formats(id),
  UNIQUE(format_id, code)
);

CREATE INDEX IF NOT EXISTS idx_cost_codes_format 
  ON cost_codes(format_id);
CREATE INDEX IF NOT EXISTS idx_cost_codes_parent 
  ON cost_codes(format_id, parent_code);

-- Comparison sessions
CREATE TABLE IF NOT EXISTS sessions (
  id TEXT PRIMARY KEY,
  format_id TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'uploading',
  progress_total INTEGER DEFAULT 0,
  progress_completed INTEGER DEFAULT 0,
  progress_current_file TEXT,
  error_message TEXT,
  created_at TEXT DEFAULT (datetime('now')),
  updated_at TEXT DEFAULT (datetime('now')),
  FOREIGN KEY (format_id) REFERENCES cost_code_formats(id)
);

CREATE INDEX IF NOT EXISTS idx_sessions_status 
  ON sessions(status);

-- Uploaded estimates
CREATE TABLE IF NOT EXISTS estimates (
  id TEXT PRIMARY KEY,
  session_id TEXT NOT NULL,
  filename TEXT NOT NULL,
  r2_key TEXT NOT NULL,
  
  -- Contractor info
  contractor_name TEXT,
  contractor_address TEXT,
  contractor_phone TEXT,
  contractor_email TEXT,
  
  -- Client info
  client_name TEXT,
  client_project_name TEXT,
  
  -- Document totals
  subtotal REAL,
  tax REAL,
  grand_total REAL,
  
  -- Processing
  status TEXT NOT NULL DEFAULT 'pending',
  error_message TEXT,
  raw_text TEXT,
  
  created_at TEXT DEFAULT (datetime('now')),
  updated_at TEXT DEFAULT (datetime('now')),
  
  FOREIGN KEY (session_id) REFERENCES sessions(id) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_estimates_session 
  ON estimates(session_id);
CREATE INDEX IF NOT EXISTS idx_estimates_status 
  ON estimates(status);

-- Extracted line items
CREATE TABLE IF NOT EXISTS line_items (
  id TEXT PRIMARY KEY,
  estimate_id TEXT NOT NULL,
  
  -- Raw extracted data
  description TEXT NOT NULL,
  section_header TEXT,
  quantity REAL,
  unit TEXT,
  unit_price REAL,
  total_price REAL NOT NULL,
  line_number INTEGER,
  
  -- Classification
  cost_code TEXT,
  confidence REAL DEFAULT 0,
  is_manual_override INTEGER DEFAULT 0,
  alternate_codes TEXT,  -- JSON array
  
  created_at TEXT DEFAULT (datetime('now')),
  updated_at TEXT DEFAULT (datetime('now')),
  
  FOREIGN KEY (estimate_id) REFERENCES estimates(id) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_line_items_estimate 
  ON line_items(estimate_id);
CREATE INDEX IF NOT EXISTS idx_line_items_cost_code 
  ON line_items(cost_code);

-- Document notes/exclusions
CREATE TABLE IF NOT EXISTS estimate_notes (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  estimate_id TEXT NOT NULL,
  note_type TEXT NOT NULL,  -- 'note', 'exclusion', 'disclaimer', 'term'
  content TEXT NOT NULL,
  FOREIGN KEY (estimate_id) REFERENCES estimates(id) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_estimate_notes_estimate 
  ON estimate_notes(estimate_id);

-- Cached comparisons (regenerated on reclassification)
CREATE TABLE IF NOT EXISTS comparisons (
  id TEXT PRIMARY KEY,
  session_id TEXT NOT NULL UNIQUE,
  comparison_data TEXT NOT NULL,  -- JSON blob
  generated_at TEXT DEFAULT (datetime('now')),
  FOREIGN KEY (session_id) REFERENCES sessions(id) ON DELETE CASCADE
);
```

### Seed Data (seed.sql)

```sql
-- Insert residential format
INSERT OR IGNORE INTO cost_code_formats (id, name, version, description)
VALUES (
  'residential-v1',
  'Residential Construction',
  '1.0',
  'Standard cost codes for residential construction projects'
);

-- Level 1: Divisions
INSERT OR IGNORE INTO cost_codes (format_id, code, label, level, parent_code, sort_order, keywords) VALUES
('residential-v1', '00', 'Demolition', 1, NULL, 0, '["demo", "tear out", "removal", "abatement", "strip", "gut"]'),
('residential-v1', '01', 'Sitework', 1, NULL, 1, '["site", "grading", "excavation", "landscaping", "paving", "utilities", "dirt"]'),
('residential-v1', '02', 'Foundation', 1, NULL, 2, '["foundation", "footing", "concrete", "slab", "basement", "crawlspace", "stem wall"]'),
('residential-v1', '03', 'Framing', 1, NULL, 3, '["framing", "lumber", "studs", "joists", "rafters", "trusses", "sheathing", "deck"]'),
('residential-v1', '04', 'Roofing', 1, NULL, 4, '["roof", "roofing", "shingle", "flashing", "gutter", "soffit", "fascia", "ridge"]'),
('residential-v1', '05', 'Windows & Doors', 1, NULL, 5, '["window", "door", "entry", "sliding", "french", "hardware", "glass", "screen"]'),
('residential-v1', '06', 'Siding & Trim', 1, NULL, 6, '["siding", "trim", "exterior", "cladding", "stucco", "brick", "hardie", "lap"]'),
('residential-v1', '07', 'Electrical', 1, NULL, 7, '["electrical", "wiring", "panel", "outlet", "switch", "lighting", "fixture", "circuit"]'),
('residential-v1', '08', 'HVAC', 1, NULL, 8, '["hvac", "heating", "cooling", "air conditioning", "furnace", "ductwork", "ventilation", "ac", "heat pump"]'),
('residential-v1', '09', 'Plumbing', 1, NULL, 9, '["plumbing", "pipe", "fixture", "water heater", "drain", "faucet", "toilet", "shower", "tub"]'),
('residential-v1', '10', 'Insulation', 1, NULL, 10, '["insulation", "batt", "blown", "spray foam", "vapor barrier", "r-value"]'),
('residential-v1', '11', 'Carpentry', 1, NULL, 11, '["carpentry", "finish", "millwork", "stairs", "railing", "built-in", "trim", "crown"]'),
('residential-v1', '12', 'Casework', 1, NULL, 12, '["cabinet", "countertop", "vanity", "casework", "storage", "granite", "quartz"]'),
('residential-v1', '13', 'Drywall & Finishes', 1, NULL, 13, '["drywall", "paint", "texture", "tile", "flooring", "finish", "hardwood", "carpet", "lvp"]'),
('residential-v1', '14', 'Specialties', 1, NULL, 14, '["appliance", "fireplace", "specialty", "mirror", "accessory", "bath accessory"]'),
('residential-v1', '15', 'General Conditions', 1, NULL, 15, '["general", "overhead", "supervision", "permits", "insurance", "cleanup", "dumpster", "temporary"]');

-- Level 2: Cost Types for each division
-- Demolition
INSERT OR IGNORE INTO cost_codes (format_id, code, label, level, parent_code, sort_order, keywords) VALUES
('residential-v1', '00_L', 'Labor', 2, '00', 0, '["labor", "hours", "crew", "workers", "man hours"]'),
('residential-v1', '00_M', 'Material', 2, '00', 1, '["material", "supplies", "equipment", "rental", "disposal"]'),
('residential-v1', '00_S', 'Subcontracts', 2, '00', 2, '["subcontract", "sub", "contractor", "hauling"]');

-- Sitework
INSERT OR IGNORE INTO cost_codes (format_id, code, label, level, parent_code, sort_order, keywords) VALUES
('residential-v1', '01_L', 'Labor', 2, '01', 0, '["labor", "hours", "crew", "workers"]'),
('residential-v1', '01_M', 'Material', 2, '01', 1, '["material", "gravel", "fill", "topsoil", "sod", "plants"]'),
('residential-v1', '01_S', 'Subcontracts', 2, '01', 2, '["subcontract", "excavation", "grading", "paving", "landscaper"]');

-- Foundation
INSERT OR IGNORE INTO cost_codes (format_id, code, label, level, parent_code, sort_order, keywords) VALUES
('residential-v1', '02_L', 'Labor', 2, '02', 0, '["labor", "hours", "crew", "workers", "pour"]'),
('residential-v1', '02_M', 'Material', 2, '02', 1, '["material", "concrete", "rebar", "forms", "anchor bolts"]'),
('residential-v1', '02_S', 'Subcontracts', 2, '02', 2, '["subcontract", "concrete contractor", "foundation sub"]');

-- Framing
INSERT OR IGNORE INTO cost_codes (format_id, code, label, level, parent_code, sort_order, keywords) VALUES
('residential-v1', '03_L', 'Labor', 2, '03', 0, '["labor", "hours", "carpenter", "framer", "crew"]'),
('residential-v1', '03_M', 'Material', 2, '03', 1, '["lumber", "plywood", "osb", "hardware", "nails", "screws", "hangers"]'),
('residential-v1', '03_S', 'Subcontracts', 2, '03', 2, '["subcontract", "framing sub", "framer", "truss"]');

-- Roofing
INSERT OR IGNORE INTO cost_codes (format_id, code, label, level, parent_code, sort_order, keywords) VALUES
('residential-v1', '04_L', 'Labor', 2, '04', 0, '["labor", "hours", "roofer", "crew"]'),
('residential-v1', '04_M', 'Material', 2, '04', 1, '["shingles", "underlayment", "flashing", "vents", "drip edge"]'),
('residential-v1', '04_S', 'Subcontracts', 2, '04', 2, '["subcontract", "roofing sub", "roofer", "gutter installer"]');

-- Windows & Doors
INSERT OR IGNORE INTO cost_codes (format_id, code, label, level, parent_code, sort_order, keywords) VALUES
('residential-v1', '05_L', 'Labor', 2, '05', 0, '["labor", "hours", "installation", "installer"]'),
('residential-v1', '05_M', 'Material', 2, '05', 1, '["windows", "doors", "hardware", "weatherstrip", "threshold"]'),
('residential-v1', '05_S', 'Subcontracts', 2, '05', 2, '["subcontract", "window installer", "door installer"]');

-- Siding & Trim
INSERT OR IGNORE INTO cost_codes (format_id, code, label, level, parent_code, sort_order, keywords) VALUES
('residential-v1', '06_L', 'Labor', 2, '06', 0, '["labor", "hours", "crew", "installer"]'),
('residential-v1', '06_M', 'Material', 2, '06', 1, '["siding", "trim", "caulk", "house wrap", "flashing"]'),
('residential-v1', '06_S', 'Subcontracts', 2, '06', 2, '["subcontract", "siding sub", "stucco contractor"]');

-- Electrical
INSERT OR IGNORE INTO cost_codes (format_id, code, label, level, parent_code, sort_order, keywords) VALUES
('residential-v1', '07_L', 'Labor', 2, '07', 0, '["labor", "hours", "electrician"]'),
('residential-v1', '07_M', 'Material', 2, '07', 1, '["wire", "panel", "breakers", "outlets", "switches", "fixtures"]'),
('residential-v1', '07_S', 'Subcontracts', 2, '07', 2, '["subcontract", "electrical sub", "electrician"]');

-- HVAC
INSERT OR IGNORE INTO cost_codes (format_id, code, label, level, parent_code, sort_order, keywords) VALUES
('residential-v1', '08_L', 'Labor', 2, '08', 0, '["labor", "hours", "hvac tech", "installer"]'),
('residential-v1', '08_M', 'Material', 2, '08', 1, '["furnace", "ac unit", "ductwork", "vents", "thermostat", "line set"]'),
('residential-v1', '08_S', 'Subcontracts', 2, '08', 2, '["subcontract", "hvac sub", "hvac contractor"]');

-- Plumbing
INSERT OR IGNORE INTO cost_codes (format_id, code, label, level, parent_code, sort_order, keywords) VALUES
('residential-v1', '09_L', 'Labor', 2, '09', 0, '["labor", "hours", "plumber"]'),
('residential-v1', '09_M', 'Material', 2, '09', 1, '["pipe", "fittings", "fixtures", "water heater", "faucets", "toilets"]'),
('residential-v1', '09_S', 'Subcontracts', 2, '09', 2, '["subcontract", "plumbing sub", "plumber"]');

-- Insulation
INSERT OR IGNORE INTO cost_codes (format_id, code, label, level, parent_code, sort_order, keywords) VALUES
('residential-v1', '10_L', 'Labor', 2, '10', 0, '["labor", "hours", "installer"]'),
('residential-v1', '10_M', 'Material', 2, '10', 1, '["batt insulation", "blown insulation", "foam", "vapor barrier"]'),
('residential-v1', '10_S', 'Subcontracts', 2, '10', 2, '["subcontract", "insulation sub", "insulator"]');

-- Carpentry
INSERT OR IGNORE INTO cost_codes (format_id, code, label, level, parent_code, sort_order, keywords) VALUES
('residential-v1', '11_L', 'Labor', 2, '11', 0, '["labor", "hours", "finish carpenter", "trim carpenter"]'),
('residential-v1', '11_M', 'Material', 2, '11', 1, '["trim", "molding", "baseboard", "casing", "crown", "hardware"]'),
('residential-v1', '11_S', 'Subcontracts', 2, '11', 2, '["subcontract", "finish sub", "stair builder"]');

-- Casework
INSERT OR IGNORE INTO cost_codes (format_id, code, label, level, parent_code, sort_order, keywords) VALUES
('residential-v1', '12_L', 'Labor', 2, '12', 0, '["labor", "hours", "installer", "cabinet installer"]'),
('residential-v1', '12_M', 'Material', 2, '12', 1, '["cabinets", "countertops", "hardware", "pulls", "hinges"]'),
('residential-v1', '12_S', 'Subcontracts', 2, '12', 2, '["subcontract", "cabinet maker", "countertop fabricator"]');

-- Drywall & Finishes
INSERT OR IGNORE INTO cost_codes (format_id, code, label, level, parent_code, sort_order, keywords) VALUES
('residential-v1', '13_L', 'Labor', 2, '13', 0, '["labor", "hours", "drywaller", "painter", "tile setter", "flooring installer"]'),
('residential-v1', '13_M', 'Material', 2, '13', 1, '["drywall", "mud", "tape", "paint", "tile", "flooring", "grout"]'),
('residential-v1', '13_S', 'Subcontracts', 2, '13', 2, '["subcontract", "drywall sub", "painter", "tile sub", "flooring sub"]');

-- Specialties
INSERT OR IGNORE INTO cost_codes (format_id, code, label, level, parent_code, sort_order, keywords) VALUES
('residential-v1', '14_L', 'Labor', 2, '14', 0, '["labor", "hours", "installer"]'),
('residential-v1', '14_M', 'Material', 2, '14', 1, '["appliances", "fireplace", "mirrors", "accessories", "bath hardware"]'),
('residential-v1', '14_S', 'Subcontracts', 2, '14', 2, '["subcontract", "appliance installer", "fireplace installer"]');

-- General Conditions
INSERT OR IGNORE INTO cost_codes (format_id, code, label, level, parent_code, sort_order, keywords) VALUES
('residential-v1', '15_L', 'Labor', 2, '15', 0, '["supervision", "project management", "cleanup labor"]'),
('residential-v1', '15_M', 'Material', 2, '15', 1, '["temporary facilities", "safety equipment", "porta potty", "dumpster"]'),
('residential-v1', '15_S', 'Subcontracts', 2, '15', 2, '["permits", "fees", "insurance", "bonds", "inspections"]');
```

## Type-Safe Query Helpers

### Database Row Types (types.ts)

```typescript
// Raw database row types (matching SQL schema)

export interface CostCodeFormatRow {
  id: string;
  name: string;
  version: string;
  description: string | null;
  created_at: string;
  updated_at: string;
}

export interface CostCodeRow {
  id: number;
  format_id: string;
  code: string;
  label: string;
  level: number;
  parent_code: string | null;
  sort_order: number;
  keywords: string | null;  // JSON string
  description: string | null;
}

export interface SessionRow {
  id: string;
  format_id: string;
  status: string;
  progress_total: number;
  progress_completed: number;
  progress_current_file: string | null;
  error_message: string | null;
  created_at: string;
  updated_at: string;
}

export interface EstimateRow {
  id: string;
  session_id: string;
  filename: string;
  r2_key: string;
  contractor_name: string | null;
  contractor_address: string | null;
  contractor_phone: string | null;
  contractor_email: string | null;
  client_name: string | null;
  client_project_name: string | null;
  subtotal: number | null;
  tax: number | null;
  grand_total: number | null;
  status: string;
  error_message: string | null;
  raw_text: string | null;
  created_at: string;
  updated_at: string;
}

export interface LineItemRow {
  id: string;
  estimate_id: string;
  description: string;
  section_header: string | null;
  quantity: number | null;
  unit: string | null;
  unit_price: number | null;
  total_price: number;
  line_number: number | null;
  cost_code: string | null;
  confidence: number;
  is_manual_override: number;  // SQLite uses 0/1 for boolean
  alternate_codes: string | null;  // JSON string
  created_at: string;
  updated_at: string;
}

export interface EstimateNoteRow {
  id: number;
  estimate_id: string;
  note_type: string;
  content: string;
}

export interface ComparisonRow {
  id: string;
  session_id: string;
  comparison_data: string;  // JSON string
  generated_at: string;
}
```

### Query Helpers (queries.ts)

```typescript
import type { D1Database } from '@cloudflare/workers-types';
import type {
  CostCode,
  Session,
  Estimate,
  LineItem,
  EstimateNote,
  Comparison,
} from '@estimate-compare/shared';
import type {
  CostCodeRow,
  SessionRow,
  EstimateRow,
  LineItemRow,
  EstimateNoteRow,
  ComparisonRow,
} from './types';

// ============ COST CODES ============

export async function getCostCodesByFormat(
  db: D1Database,
  formatId: string
): Promise<CostCode[]> {
  const result = await db
    .prepare(`
      SELECT code, label, level, parent_code, sort_order, keywords, description
      FROM cost_codes
      WHERE format_id = ?
      ORDER BY sort_order, code
    `)
    .bind(formatId)
    .all<CostCodeRow>();

  return result.results.map(row => ({
    code: row.code,
    label: row.label,
    level: row.level,
    parentCode: row.parent_code,
    sortOrder: row.sort_order,
    keywords: row.keywords ? JSON.parse(row.keywords) : undefined,
    description: row.description || undefined,
  }));
}

// ============ SESSIONS ============

export async function getSession(
  db: D1Database,
  sessionId: string
): Promise<Session | null> {
  const row = await db
    .prepare('SELECT * FROM sessions WHERE id = ?')
    .bind(sessionId)
    .first<SessionRow>();

  if (!row) return null;

  return mapSessionRow(row);
}

export async function createSession(
  db: D1Database,
  sessionId: string,
  formatId: string
): Promise<void> {
  await db
    .prepare(`
      INSERT INTO sessions (id, format_id, status)
      VALUES (?, ?, 'uploading')
    `)
    .bind(sessionId, formatId)
    .run();
}

export async function updateSessionStatus(
  db: D1Database,
  sessionId: string,
  status: Session['status'],
  progress?: { total?: number; completed?: number; currentFile?: string | null },
  errorMessage?: string
): Promise<void> {
  const updates: string[] = ['status = ?', "updated_at = datetime('now')"];
  const params: (string | number | null)[] = [status];

  if (progress?.total !== undefined) {
    updates.push('progress_total = ?');
    params.push(progress.total);
  }
  if (progress?.completed !== undefined) {
    updates.push('progress_completed = ?');
    params.push(progress.completed);
  }
  if (progress?.currentFile !== undefined) {
    updates.push('progress_current_file = ?');
    params.push(progress.currentFile);
  }
  if (errorMessage !== undefined) {
    updates.push('error_message = ?');
    params.push(errorMessage);
  }

  params.push(sessionId);

  await db
    .prepare(`UPDATE sessions SET ${updates.join(', ')} WHERE id = ?`)
    .bind(...params)
    .run();
}

// ============ ESTIMATES ============

export async function getEstimatesBySession(
  db: D1Database,
  sessionId: string
): Promise<Estimate[]> {
  const estimates = await db
    .prepare('SELECT * FROM estimates WHERE session_id = ? ORDER BY created_at')
    .bind(sessionId)
    .all<EstimateRow>();

  const result: Estimate[] = [];

  for (const estRow of estimates.results) {
    const lineItems = await getLineItemsByEstimate(db, estRow.id);
    const notes = await getNotesByEstimate(db, estRow.id);
    result.push(mapEstimateRow(estRow, lineItems, notes));
  }

  return result;
}

export async function getEstimate(
  db: D1Database,
  estimateId: string
): Promise<Estimate | null> {
  const row = await db
    .prepare('SELECT * FROM estimates WHERE id = ?')
    .bind(estimateId)
    .first<EstimateRow>();

  if (!row) return null;

  const lineItems = await getLineItemsByEstimate(db, estimateId);
  const notes = await getNotesByEstimate(db, estimateId);

  return mapEstimateRow(row, lineItems, notes);
}

export async function createEstimate(
  db: D1Database,
  estimate: {
    id: string;
    sessionId: string;
    filename: string;
    r2Key: string;
  }
): Promise<void> {
  await db
    .prepare(`
      INSERT INTO estimates (id, session_id, filename, r2_key, status)
      VALUES (?, ?, ?, ?, 'pending')
    `)
    .bind(estimate.id, estimate.sessionId, estimate.filename, estimate.r2Key)
    .run();
}

export async function updateEstimate(
  db: D1Database,
  estimateId: string,
  data: Partial<{
    contractorName: string | null;
    contractorAddress: string | null;
    contractorPhone: string | null;
    contractorEmail: string | null;
    clientName: string | null;
    clientProjectName: string | null;
    subtotal: number | null;
    tax: number | null;
    grandTotal: number | null;
    status: string;
    errorMessage: string | null;
    rawText: string | null;
  }>
): Promise<void> {
  const fieldMap: Record<string, string> = {
    contractorName: 'contractor_name',
    contractorAddress: 'contractor_address',
    contractorPhone: 'contractor_phone',
    contractorEmail: 'contractor_email',
    clientName: 'client_name',
    clientProjectName: 'client_project_name',
    subtotal: 'subtotal',
    tax: 'tax',
    grandTotal: 'grand_total',
    status: 'status',
    errorMessage: 'error_message',
    rawText: 'raw_text',
  };

  const updates: string[] = ["updated_at = datetime('now')"];
  const params: (string | number | null)[] = [];

  for (const [key, value] of Object.entries(data)) {
    if (value !== undefined && fieldMap[key]) {
      updates.push(`${fieldMap[key]} = ?`);
      params.push(value);
    }
  }

  params.push(estimateId);

  await db
    .prepare(`UPDATE estimates SET ${updates.join(', ')} WHERE id = ?`)
    .bind(...params)
    .run();
}

// ============ LINE ITEMS ============

export async function getLineItemsByEstimate(
  db: D1Database,
  estimateId: string
): Promise<LineItem[]> {
  const result = await db
    .prepare(`
      SELECT * FROM line_items 
      WHERE estimate_id = ? 
      ORDER BY line_number, id
    `)
    .bind(estimateId)
    .all<LineItemRow>();

  return result.results.map(mapLineItemRow);
}

export async function createLineItem(
  db: D1Database,
  item: {
    id: string;
    estimateId: string;
    description: string;
    sectionHeader: string | null;
    quantity: number | null;
    unit: string | null;
    unitPrice: number | null;
    totalPrice: number;
    lineNumber: number | null;
    costCode: string | null;
    confidence: number;
    alternateCodes: Array<{ code: string; confidence: number }>;
  }
): Promise<void> {
  await db
    .prepare(`
      INSERT INTO line_items (
        id, estimate_id, description, section_header,
        quantity, unit, unit_price, total_price, line_number,
        cost_code, confidence, alternate_codes
      )
      VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
    `)
    .bind(
      item.id,
      item.estimateId,
      item.description,
      item.sectionHeader,
      item.quantity,
      item.unit,
      item.unitPrice,
      item.totalPrice,
      item.lineNumber,
      item.costCode,
      item.confidence,
      JSON.stringify(item.alternateCodes)
    )
    .run();
}

export async function updateLineItemClassification(
  db: D1Database,
  lineItemId: string,
  costCode: string,
  isManualOverride: boolean = true
): Promise<void> {
  await db
    .prepare(`
      UPDATE line_items 
      SET cost_code = ?, 
          is_manual_override = ?,
          confidence = 1.0,
          updated_at = datetime('now')
      WHERE id = ?
    `)
    .bind(costCode, isManualOverride ? 1 : 0, lineItemId)
    .run();
}

// ============ NOTES ============

export async function getNotesByEstimate(
  db: D1Database,
  estimateId: string
): Promise<EstimateNote[]> {
  const result = await db
    .prepare('SELECT * FROM estimate_notes WHERE estimate_id = ?')
    .bind(estimateId)
    .all<EstimateNoteRow>();

  return result.results.map(row => ({
    id: row.id,
    noteType: row.note_type as EstimateNote['noteType'],
    content: row.content,
  }));
}

export async function createNote(
  db: D1Database,
  estimateId: string,
  noteType: string,
  content: string
): Promise<void> {
  await db
    .prepare(`
      INSERT INTO estimate_notes (estimate_id, note_type, content)
      VALUES (?, ?, ?)
    `)
    .bind(estimateId, noteType, content)
    .run();
}

// ============ COMPARISONS ============

export async function getComparison(
  db: D1Database,
  sessionId: string
): Promise<Comparison | null> {
  const row = await db
    .prepare('SELECT * FROM comparisons WHERE session_id = ?')
    .bind(sessionId)
    .first<ComparisonRow>();

  if (!row) return null;

  return JSON.parse(row.comparison_data) as Comparison;
}

export async function saveComparison(
  db: D1Database,
  comparison: Comparison
): Promise<void> {
  await db
    .prepare(`
      INSERT OR REPLACE INTO comparisons (id, session_id, comparison_data, generated_at)
      VALUES (?, ?, ?, datetime('now'))
    `)
    .bind(
      comparison.id,
      comparison.sessionId,
      JSON.stringify(comparison)
    )
    .run();
}

// ============ FULL SESSION DATA ============

export async function getSessionWithEstimates(
  db: D1Database,
  sessionId: string
): Promise<{
  session: Session | null;
  estimates: Estimate[];
  comparison: Comparison | null;
}> {
  const session = await getSession(db, sessionId);
  if (!session) {
    return { session: null, estimates: [], comparison: null };
  }

  const estimates = await getEstimatesBySession(db, sessionId);
  const comparison = await getComparison(db, sessionId);

  return { session, estimates, comparison };
}

// ============ MAPPERS ============

function mapSessionRow(row: SessionRow): Session {
  return {
    id: row.id,
    formatId: row.format_id,
    status: row.status as Session['status'],
    progress: {
      total: row.progress_total,
      completed: row.progress_completed,
      currentFile: row.progress_current_file || undefined,
    },
    errorMessage: row.error_message || undefined,
    createdAt: row.created_at,
    updatedAt: row.updated_at,
  };
}

function mapEstimateRow(
  row: EstimateRow,
  lineItems: LineItem[],
  notes: EstimateNote[]
): Estimate {
  return {
    id: row.id,
    sessionId: row.session_id,
    filename: row.filename,
    contractorName: row.contractor_name,
    grandTotal: row.grand_total,
    status: row.status as Estimate['status'],
    errorMessage: row.error_message || undefined,
    contractor: {
      name: row.contractor_name,
      address: row.contractor_address,
      phone: row.contractor_phone,
      email: row.contractor_email,
    },
    client: row.client_name ? {
      name: row.client_name,
      projectName: row.client_project_name,
    } : null,
    documentTotals: {
      subtotal: row.subtotal,
      tax: row.tax,
      grandTotal: row.grand_total,
    },
    lineItems,
    notes,
    rawText: row.raw_text || undefined,
  };
}

function mapLineItemRow(row: LineItemRow): LineItem {
  return {
    id: row.id,
    estimateId: row.estimate_id,
    description: row.description,
    sectionHeader: row.section_header,
    quantity: row.quantity,
    unit: row.unit,
    unitPrice: row.unit_price,
    totalPrice: row.total_price,
    lineNumber: row.line_number,
    costCode: row.cost_code,
    confidence: row.confidence,
    isManualOverride: row.is_manual_override === 1,
    alternateCodes: row.alternate_codes 
      ? JSON.parse(row.alternate_codes) 
      : [],
  };
}
```

## Todo List

### Schema Design

- [ ] Create schema.sql with all tables
- [ ] Add foreign key constraints
- [ ] Add appropriate indexes
- [ ] Verify CASCADE delete works correctly
- [ ] Test schema with SQLite locally

### Seed Data

- [ ] Create seed.sql with cost code format
- [ ] Add all 16 divisions (level 1 codes)
- [ ] Add all 48 cost types (level 2 codes: 16 × 3)
- [ ] Add comprehensive keywords for each code
- [ ] Verify seed data loads correctly

### Query Helpers

- [ ] Create db/types.ts with row types
- [ ] Implement getCostCodesByFormat
- [ ] Implement getSession
- [ ] Implement createSession
- [ ] Implement updateSessionStatus
- [ ] Implement getEstimatesBySession
- [ ] Implement getEstimate
- [ ] Implement createEstimate
- [ ] Implement updateEstimate
- [ ] Implement getLineItemsByEstimate
- [ ] Implement createLineItem
- [ ] Implement updateLineItemClassification
- [ ] Implement getNotesByEstimate
- [ ] Implement createNote
- [ ] Implement getComparison
- [ ] Implement saveComparison
- [ ] Implement getSessionWithEstimates
- [ ] Create mapper functions for all entities

### Testing

- [ ] Test schema creation
- [ ] Test seed data insertion
- [ ] Test all query helpers with mock data
- [ ] Test cascade deletes
- [ ] Verify JSON serialization/deserialization
- [ ] Test with D1 local emulator

### Migration Scripts

- [ ] Document migration process
- [ ] Create migration helper commands in package.json
- [ ] Test migration from fresh state
- [ ] Test migration from existing state

## Verification Checklist

- [ ] Schema executes without errors
- [ ] Seed data loads correctly
- [ ] All 64 cost codes present (16 divisions + 48 cost types)
- [ ] Foreign key constraints work
- [ ] Indexes created
- [ ] Query helpers compile without type errors
- [ ] All queries execute successfully
- [ ] JSON fields parse correctly

## Notes

### D1 SQLite Differences

- Uses TEXT for datetime (not native datetime type)
- Boolean stored as INTEGER (0/1)
- No native JSON type (use TEXT with JSON functions)
- AUTOINCREMENT only on INTEGER PRIMARY KEY

### Performance Considerations

- Keep JSON columns small (alternate_codes, keywords)
- Use prepared statements (all queries do)
- Consider batching line item inserts for large estimates
- Index columns used in WHERE clauses

### Migration Strategy

For MVP, use destructive migrations (drop and recreate). For production:

```bash
# Backup current data
wrangler d1 export estimate-compare > backup.sql

# Apply migration
wrangler d1 execute estimate-compare --file=./migrations/002_add_field.sql
```

## Time Estimate

| Task | Estimate |
|------|----------|
| Schema design | 30 min |
| Schema DDL | 30 min |
| Seed data | 45 min |
| Row types | 30 min |
| Query helpers | 2 hours |
| Testing | 45 min |
| **Total** | **~5 hours** |
