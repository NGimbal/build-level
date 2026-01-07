# Module 02: Shared Package

## Overview

The shared package contains TypeScript types, interfaces, and utility functions used by both frontend and backend packages. This ensures type consistency across the entire application.

## Goals

- Define all shared TypeScript interfaces
- Create generic cost code format schema
- Implement shared utility functions
- Ensure proper module exports for consumption

## File Structure

```
packages/shared/
├── package.json
├── tsconfig.json
└── src/
    ├── index.ts              # Main export file
    ├── types/
    │   ├── index.ts          # Type exports
    │   ├── cost-codes.ts     # Cost code format types
    │   ├── session.ts        # Session types
    │   ├── estimate.ts       # Estimate types
    │   ├── line-item.ts      # Line item types
    │   └── comparison.ts     # Comparison types
    ├── constants/
    │   ├── index.ts
    │   └── cost-codes.ts     # Default cost code format
    └── utils/
        ├── index.ts
        ├── formatting.ts     # Currency, number formatting
        └── validation.ts     # Shared validation helpers
```

## Type Definitions

### Cost Code Types (src/types/cost-codes.ts)

```typescript
/**
 * A cost code classification format (e.g., Residential, MasterFormat, UniFormat)
 * Designed to be format-agnostic and support any hierarchical cost code system
 */
export interface CostCodeFormat {
  /** Unique identifier for the format */
  id: string;
  /** Display name */
  name: string;
  /** Version string */
  version: string;
  /** Optional description */
  description?: string;
}

/**
 * Individual cost code within a format
 */
export interface CostCode {
  /** Unique code identifier (e.g., "03", "03_L") */
  code: string;
  /** Display label (e.g., "Framing", "Labor") */
  label: string;
  /** Hierarchy level (1 = division, 2 = cost type, etc.) */
  level: number;
  /** Parent code reference, null for root level */
  parentCode: string | null;
  /** Display order within siblings - use fractional indexing */
  sortOrder: string;
  /** Optional keywords for classification matching */
  keywords?: string[];
  /** Optional longer description */
  description?: string;
}

/**
 * Cost code with computed full label path
 */
export interface CostCodeWithPath extends CostCode {
  /** Full label path (e.g., "Framing → Labor") */
  fullLabel: string;
}
```

### Session Types (src/types/session.ts)

```typescript
export type SessionStatus = "uploading" | "processing" | "ready" | "error";

export interface SessionProgress {
  total: number;
  completed: number;
  currentFile?: string;
}

export interface Session {
  id: string;
  formatId: string;
  status: SessionStatus;
  progress?: SessionProgress;
  errorMessage?: string;
  createdAt: string;
  updatedAt: string;
}

export interface SessionWithEstimates extends Session {
  estimates: EstimateHeader[];
}
```

### Estimate Types (src/types/estimate.ts)

```typescript
export type EstimateStatus =
  | "pending"
  | "extracting"
  | "classifying"
  | "ready"
  | "error";

export interface Contractor {
  name: string | null;
  address: string | null;
  phone: string | null;
  email: string | null;
}

export interface Client {
  name: string | null;
  projectName: string | null;
}

export interface DocumentTotals {
  subtotal: number | null;
  tax: number | null;
  grandTotal: number | null;
}

export interface EstimateHeader {
  id: string;
  sessionId: string;
  filename: string;
  contractorName: string | null;
  grandTotal: number | null;
  status: EstimateStatus;
  errorMessage?: string;
}

export interface Estimate extends EstimateHeader {
  contractor: Contractor;
  client: Client | null;
  documentTotals: DocumentTotals;
  lineItems: LineItem[];
  notes: EstimateNote[];
  rawText?: string;
}

export interface EstimateNote {
  id: number;
  noteType: "note" | "exclusion" | "disclaimer" | "term";
  content: string;
}
```

### Line Item Types (src/types/line-item.ts)

```typescript
export interface LineItemClassification {
  code: string | null;
  confidence: number;
  isManualOverride: boolean;
  alternateCodes: AlternateCode[];
}

export interface AlternateCode {
  code: string;
  confidence: number;
}

export interface LineItem {
  id: string;
  estimateId: string;

  // Raw extracted data
  description: string;
  sectionHeader: string | null;
  quantity: number | null;
  unit: string | null;
  unitPrice: number | null;
  totalPrice: number;
  lineNumber: number | null;

  // Classification
  costCode: string | null;
  confidence: number;
  isManualOverride: boolean;
  alternateCodes: AlternateCode[];
}

/**
 * Line item as returned from LLM parsing (before classification)
 */
export interface ParsedLineItem {
  description: string;
  sectionHeader: string | null;
  quantity: number | null;
  unit: string | null;
  unitPrice: number | null;
  totalPrice: number;
  lineNumber: number | null;
}
```

### Comparison Types (src/types/comparison.ts)

```typescript
export interface ComparisonCell {
  total: number;
  lineItemIds: string[];
  lineItemCount: number;
}

export interface ComputedStats {
  min: number | null;
  max: number | null;
  average: number | null;
  range: number | null;
  rangePercent: number | null;
}

export interface ComparisonRow {
  code: string;
  label: string;
  fullLabel: string;
  level: number;
  isSubtotal: boolean;

  /** Per-estimate amounts (estimateId -> cell) */
  amounts: Record<string, ComparisonCell | null>;

  /** Computed statistics */
  computed: ComputedStats;
}

export interface ComparisonSummary {
  byEstimate: Record<string, number>;
  range: {
    min: number;
    max: number;
    spread: number;
  };
  average: number;
}

export interface Comparison {
  id: string;
  sessionId: string;
  generatedAt: string;
  rows: ComparisonRow[];
  summary: ComparisonSummary;
}
```

## Utility Functions

### Formatting (src/utils/formatting.ts)

```typescript
/**
 * Format a number as currency
 */
export function formatCurrency(
  value: number | null | undefined,
  options?: {
    currency?: string;
    locale?: string;
    minimumFractionDigits?: number;
    maximumFractionDigits?: number;
  }
): string {
  if (value === null || value === undefined) {
    return "—";
  }

  const {
    currency = "USD",
    locale = "en-US",
    minimumFractionDigits = 0,
    maximumFractionDigits = 0,
  } = options || {};

  return value.toLocaleString(locale, {
    style: "currency",
    currency,
    minimumFractionDigits,
    maximumFractionDigits,
  });
}

/**
 * Format a number as percentage
 */
export function formatPercent(
  value: number | null | undefined,
  options?: {
    locale?: string;
    minimumFractionDigits?: number;
    maximumFractionDigits?: number;
  }
): string {
  if (value === null || value === undefined) {
    return "—";
  }

  const {
    locale = "en-US",
    minimumFractionDigits = 1,
    maximumFractionDigits = 1,
  } = options || {};

  return (
    (value * 100).toLocaleString(locale, {
      minimumFractionDigits,
      maximumFractionDigits,
    }) + "%"
  );
}

/**
 * Format a number with thousand separators
 */
export function formatNumber(
  value: number | null | undefined,
  options?: {
    locale?: string;
    minimumFractionDigits?: number;
    maximumFractionDigits?: number;
  }
): string {
  if (value === null || value === undefined) {
    return "—";
  }

  const {
    locale = "en-US",
    minimumFractionDigits = 0,
    maximumFractionDigits = 2,
  } = options || {};

  return value.toLocaleString(locale, {
    minimumFractionDigits,
    maximumFractionDigits,
  });
}
```

### Validation (src/utils/validation.ts)

```typescript
/**
 * Check if a value is a valid positive number
 */
export function isValidAmount(value: unknown): value is number {
  return typeof value === "number" && !isNaN(value) && value >= 0;
}

/**
 * Check if a string is non-empty after trimming
 */
export function isNonEmptyString(value: unknown): value is string {
  return typeof value === "string" && value.trim().length > 0;
}

/**
 * Validate a cost code format
 */
export function isValidCostCode(code: string, validCodes: string[]): boolean {
  return validCodes.includes(code);
}

/**
 * Parse a monetary string to number
 * Handles formats like "$1,234.56", "1234.56", "(1,234.56)"
 */
export function parseMonetaryValue(value: string): number | null {
  if (!value || typeof value !== "string") {
    return null;
  }

  // Remove currency symbols, spaces, and commas
  let cleaned = value.replace(/[$,\s]/g, "");

  // Handle negative values in parentheses
  const isNegative = cleaned.startsWith("(") && cleaned.endsWith(")");
  if (isNegative) {
    cleaned = cleaned.slice(1, -1);
  }

  const parsed = parseFloat(cleaned);

  if (isNaN(parsed)) {
    return null;
  }

  return isNegative ? -parsed : parsed;
}
```

## Default Cost Code Format

### Constants (src/constants/cost-codes.ts)

```typescript
import type { CostCodeFormat, CostCode } from "../types";

export const RESIDENTIAL_FORMAT: CostCodeFormat = {
  id: "residential-v1",
  name: "Residential Construction",
  version: "1.0",
  description: "Standard cost codes for residential construction projects",
};

export const RESIDENTIAL_CODES: CostCode[] = [
  // Level 1: Divisions
  {
    code: "00",
    label: "Demolition",
    level: 1,
    parentCode: null,
    sortOrder: 0,
    keywords: ["demo", "tear out", "removal", "abatement"],
  },
  {
    code: "01",
    label: "Sitework",
    level: 1,
    parentCode: null,
    sortOrder: 1,
    keywords: ["site", "grading", "excavation", "landscaping"],
  },
  {
    code: "02",
    label: "Foundation",
    level: 1,
    parentCode: null,
    sortOrder: 2,
    keywords: ["foundation", "footing", "concrete", "slab"],
  },
  {
    code: "03",
    label: "Framing",
    level: 1,
    parentCode: null,
    sortOrder: 3,
    keywords: ["framing", "lumber", "studs", "joists", "rafters"],
  },
  {
    code: "04",
    label: "Roofing",
    level: 1,
    parentCode: null,
    sortOrder: 4,
    keywords: ["roof", "shingle", "flashing", "gutter"],
  },
  {
    code: "05",
    label: "Windows & Doors",
    level: 1,
    parentCode: null,
    sortOrder: 5,
    keywords: ["window", "door", "entry", "sliding"],
  },
  {
    code: "06",
    label: "Siding & Trim",
    level: 1,
    parentCode: null,
    sortOrder: 6,
    keywords: ["siding", "trim", "exterior", "cladding"],
  },
  {
    code: "07",
    label: "Electrical",
    level: 1,
    parentCode: null,
    sortOrder: 7,
    keywords: ["electrical", "wiring", "panel", "outlet"],
  },
  {
    code: "08",
    label: "HVAC",
    level: 1,
    parentCode: null,
    sortOrder: 8,
    keywords: ["hvac", "heating", "cooling", "furnace", "ductwork"],
  },
  {
    code: "09",
    label: "Plumbing",
    level: 1,
    parentCode: null,
    sortOrder: 9,
    keywords: ["plumbing", "pipe", "fixture", "water heater"],
  },
  {
    code: "10",
    label: "Insulation",
    level: 1,
    parentCode: null,
    sortOrder: 10,
    keywords: ["insulation", "batt", "blown", "spray foam"],
  },
  {
    code: "11",
    label: "Carpentry",
    level: 1,
    parentCode: null,
    sortOrder: 11,
    keywords: ["carpentry", "finish", "millwork", "stairs"],
  },
  {
    code: "12",
    label: "Casework",
    level: 1,
    parentCode: null,
    sortOrder: 12,
    keywords: ["cabinet", "countertop", "vanity", "casework"],
  },
  {
    code: "13",
    label: "Drywall & Finishes",
    level: 1,
    parentCode: null,
    sortOrder: 13,
    keywords: ["drywall", "paint", "texture", "tile", "flooring"],
  },
  {
    code: "14",
    label: "Specialties",
    level: 1,
    parentCode: null,
    sortOrder: 14,
    keywords: ["appliance", "fireplace", "specialty", "mirror"],
  },
  {
    code: "15",
    label: "General Conditions",
    level: 1,
    parentCode: null,
    sortOrder: 15,
    keywords: ["general", "overhead", "supervision", "permits"],
  },

  // Level 2: Cost Types (Labor, Material, Subcontracts)
  // Demolition
  {
    code: "00_L",
    label: "Labor",
    level: 2,
    parentCode: "00",
    sortOrder: 0,
    keywords: ["labor", "hours", "crew"],
  },
  {
    code: "00_M",
    label: "Material",
    level: 2,
    parentCode: "00",
    sortOrder: 1,
    keywords: ["material", "supplies", "equipment"],
  },
  {
    code: "00_S",
    label: "Subcontracts",
    level: 2,
    parentCode: "00",
    sortOrder: 2,
    keywords: ["subcontract", "sub", "contractor"],
  },

  // ... (repeat for all 16 divisions)
  // See full list in seed.sql
];

/**
 * Get all codes for a specific format
 */
export function getDefaultCodes(formatId: string): CostCode[] {
  if (formatId === "residential-v1") {
    return RESIDENTIAL_CODES;
  }
  return [];
}

/**
 * Build a full label for a cost code including parent path
 */
export function buildFullLabel(code: CostCode, allCodes: CostCode[]): string {
  if (!code.parentCode) {
    return code.label;
  }

  const parent = allCodes.find((c) => c.code === code.parentCode);
  if (!parent) {
    return code.label;
  }

  return `${parent.label} → ${code.label}`;
}
```

## Main Export (src/index.ts)

```typescript
// Types
export * from "./types";

// Constants
export * from "./constants";

// Utils
export * from "./utils";
```

## Package Configuration

### package.json

```json
{
  "name": "@build-level/shared",
  "version": "0.0.1",
  "private": true,
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "exports": {
    ".": {
      "import": "./src/index.ts",
      "types": "./src/index.ts"
    },
    "./types": {
      "import": "./src/types/index.ts",
      "types": "./src/types/index.ts"
    },
    "./utils": {
      "import": "./src/utils/index.ts",
      "types": "./src/utils/index.ts"
    },
    "./constants": {
      "import": "./src/constants/index.ts",
      "types": "./src/constants/index.ts"
    }
  },
  "scripts": {
    "typecheck": "tsc --noEmit"
  },
  "devDependencies": {
    "typescript": "^5.3.3"
  }
}
```

### tsconfig.json

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"]
}
```

## Todo List

### Type Definitions

- [ ] Create src/types directory structure
- [ ] Define CostCodeFormat interface
- [ ] Define CostCode interface
- [ ] Define Session types (Session, SessionStatus, SessionProgress)
- [ ] Define EstimateHeader interface
- [ ] Define full Estimate interface
- [ ] Define Contractor interface
- [ ] Define Client interface
- [ ] Define DocumentTotals interface
- [ ] Define LineItem interface
- [ ] Define ParsedLineItem interface
- [ ] Define AlternateCode interface
- [ ] Define EstimateNote interface
- [ ] Define Comparison interface
- [ ] Define ComparisonRow interface
- [ ] Define ComparisonCell interface
- [ ] Define ComputedStats interface
- [ ] Define ComparisonSummary interface
- [ ] Create types/index.ts exporting all types

### Constants

- [ ] Create src/constants directory
- [ ] Define RESIDENTIAL_FORMAT constant
- [ ] Define complete RESIDENTIAL_CODES array (16 divisions × 3 cost types = 64 codes)
- [ ] Implement getDefaultCodes function
- [ ] Implement buildFullLabel helper
- [ ] Create constants/index.ts

### Utility Functions

- [ ] Create src/utils directory
- [ ] Implement formatCurrency function
- [ ] Implement formatPercent function
- [ ] Implement formatNumber function
- [ ] Implement isValidAmount validation
- [ ] Implement isNonEmptyString validation
- [ ] Implement isValidCostCode validation
- [ ] Implement parseMonetaryValue parser
- [ ] Create utils/index.ts

### Package Setup

- [ ] Create package.json with exports
- [ ] Create tsconfig.json
- [ ] Create main src/index.ts
- [ ] Verify imports work from backend package
- [ ] Verify imports work from frontend package

### Testing

- [ ] Verify formatCurrency handles edge cases
- [ ] Verify parseMonetaryValue handles various formats
- [ ] Verify type exports are accessible
- [ ] Verify no circular dependencies

## Verification Checklist

- [ ] `pnpm typecheck` passes in shared package
- [ ] Backend can import `@build-level/shared`
- [ ] Frontend can import `@build-level/shared`
- [ ] All 64 cost codes are defined correctly
- [ ] Parent-child relationships are correct
- [ ] Utility functions handle null/undefined

## Notes

### Import Best Practices

```typescript
// Import specific types
import type { Session, Estimate, LineItem } from "@build-level/shared";

// Import specific utilities
import { formatCurrency, formatPercent } from "@build-level/shared/utils";

// Import constants
import { RESIDENTIAL_CODES } from "@build-level/shared/constants";
```

### Type Guards

Consider adding type guards for runtime validation:

```typescript
export function isEstimate(obj: unknown): obj is Estimate {
  return (
    typeof obj === "object" && obj !== null && "id" in obj && "lineItems" in obj
  );
}
```

## Time Estimate

| Task              | Estimate     |
| ----------------- | ------------ |
| Type definitions  | 1 hour       |
| Constants         | 30 min       |
| Utility functions | 45 min       |
| Package setup     | 15 min       |
| Testing           | 30 min       |
| **Total**         | **~3 hours** |
