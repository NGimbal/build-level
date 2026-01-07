# Module 07: Comparison Engine

## Overview

The comparison engine aggregates classified line items from multiple estimates and generates a unified comparison report with statistics. It produces the structured data that powers the comparison table UI.

## Goals

- Aggregate line items by cost code across estimates
- Build hierarchical comparison rows (divisions + cost types)
- Calculate statistics (min, max, average, spread)
- Generate subtotals by division
- Identify missing items and outliers
- Support regeneration on reclassification

## File Structure

```
packages/backend/src/services/
├── comparison.ts         # Main comparison generator
└── statistics.ts         # Statistical helper functions
```

## Core Implementation

### Comparison Generator (comparison.ts)

```typescript
import type {
  Comparison,
  ComparisonRow,
  ComparisonCell,
  ComparisonSummary,
  ComputedStats,
  CostCode,
  Estimate,
  LineItem,
} from '@estimate-compare/shared';

export class ComparisonGenerator {
  constructor(private costCodes: CostCode[]) {}

  /**
   * Generate a comparison from multiple estimates
   */
  generate(estimates: Estimate[]): Comparison {
    if (estimates.length === 0) {
      return this.createEmptyComparison();
    }

    const estimateIds = estimates.map(e => e.id);
    const sessionId = estimates[0].sessionId;

    // Step 1: Aggregate amounts by cost code
    const codeAmounts = this.aggregateByCode(estimates);

    // Step 2: Build comparison rows with hierarchy
    const rows = this.buildRows(codeAmounts, estimateIds);

    // Step 3: Calculate overall summary
    const summary = this.calculateSummary(estimates, codeAmounts);

    return {
      id: crypto.randomUUID(),
      sessionId,
      generatedAt: new Date().toISOString(),
      rows,
      summary,
    };
  }

  /**
   * Regenerate comparison for a session (after reclassification)
   */
  async regenerate(
    sessionId: string,
    estimates: Estimate[]
  ): Promise<Comparison> {
    return this.generate(estimates);
  }

  /**
   * Aggregate line item amounts by cost code for each estimate
   */
  private aggregateByCode(
    estimates: Estimate[]
  ): Map<string, Record<string, ComparisonCell>> {
    const codeAmounts = new Map<string, Record<string, ComparisonCell>>();

    for (const estimate of estimates) {
      for (const item of estimate.lineItems) {
        // Use cost code or default to "unclassified"
        const code = item.costCode || 'UNCLASSIFIED';

        // Initialize code entry if needed
        if (!codeAmounts.has(code)) {
          codeAmounts.set(code, {});
        }

        const byEstimate = codeAmounts.get(code)!;

        // Initialize estimate entry if needed
        if (!byEstimate[estimate.id]) {
          byEstimate[estimate.id] = {
            total: 0,
            lineItemIds: [],
            lineItemCount: 0,
          };
        }

        // Accumulate
        byEstimate[estimate.id].total += item.totalPrice;
        byEstimate[estimate.id].lineItemIds.push(item.id);
        byEstimate[estimate.id].lineItemCount += 1;
      }
    }

    return codeAmounts;
  }

  /**
   * Build comparison rows with proper hierarchy
   */
  private buildRows(
    codeAmounts: Map<string, Record<string, ComparisonCell>>,
    estimateIds: string[]
  ): ComparisonRow[] {
    const rows: ComparisonRow[] = [];

    // Get level 1 codes (divisions) sorted
    const level1Codes = this.costCodes
      .filter(c => c.level === 1)
      .sort((a, b) => a.sortOrder - b.sortOrder);

    for (const division of level1Codes) {
      const divisionRows = this.buildDivisionRows(
        division,
        codeAmounts,
        estimateIds
      );

      if (divisionRows.length > 0) {
        rows.push(...divisionRows);
      }
    }

    // Handle unclassified items
    const unclassifiedAmounts = codeAmounts.get('UNCLASSIFIED');
    if (unclassifiedAmounts && Object.keys(unclassifiedAmounts).length > 0) {
      rows.push({
        code: 'UNCLASSIFIED',
        label: 'Unclassified',
        fullLabel: 'Unclassified Items',
        level: 1,
        isSubtotal: false,
        amounts: this.mapAmountsToEstimates(unclassifiedAmounts, estimateIds),
        computed: this.computeStats(unclassifiedAmounts, estimateIds),
      });
    }

    // Add grand total row
    rows.push(this.buildGrandTotalRow(estimateIds, codeAmounts));

    return rows;
  }

  /**
   * Build rows for a single division (level 1 code)
   */
  private buildDivisionRows(
    division: CostCode,
    codeAmounts: Map<string, Record<string, ComparisonCell>>,
    estimateIds: string[]
  ): ComparisonRow[] {
    const rows: ComparisonRow[] = [];
    const divisionTotals: Record<string, ComparisonCell> = {};
    let hasData = false;

    // Get child codes (cost types) for this division
    const children = this.costCodes
      .filter(c => c.parentCode === division.code)
      .sort((a, b) => a.sortOrder - b.sortOrder);

    // Add rows for each cost type with data
    for (const child of children) {
      const amounts = codeAmounts.get(child.code);

      if (!amounts || Object.keys(amounts).length === 0) {
        continue;
      }

      // Check if any estimate has data
      const hasValues = Object.values(amounts).some(cell => cell.total > 0);
      if (!hasValues) {
        continue;
      }

      hasData = true;

      // Add to division totals
      for (const [estId, cell] of Object.entries(amounts)) {
        if (!divisionTotals[estId]) {
          divisionTotals[estId] = {
            total: 0,
            lineItemIds: [],
            lineItemCount: 0,
          };
        }
        divisionTotals[estId].total += cell.total;
        divisionTotals[estId].lineItemIds.push(...cell.lineItemIds);
        divisionTotals[estId].lineItemCount += cell.lineItemCount;
      }

      // Add detail row
      rows.push({
        code: child.code,
        label: child.label,
        fullLabel: `${division.label} → ${child.label}`,
        level: 2,
        isSubtotal: false,
        amounts: this.mapAmountsToEstimates(amounts, estimateIds),
        computed: this.computeStats(amounts, estimateIds),
      });
    }

    // Check for items directly on division code (not assigned to cost type)
    const divisionDirectAmounts = codeAmounts.get(division.code);
    if (divisionDirectAmounts && Object.keys(divisionDirectAmounts).length > 0) {
      const hasValues = Object.values(divisionDirectAmounts).some(c => c.total > 0);
      
      if (hasValues) {
        hasData = true;

        // Add to division totals
        for (const [estId, cell] of Object.entries(divisionDirectAmounts)) {
          if (!divisionTotals[estId]) {
            divisionTotals[estId] = {
              total: 0,
              lineItemIds: [],
              lineItemCount: 0,
            };
          }
          divisionTotals[estId].total += cell.total;
          divisionTotals[estId].lineItemIds.push(...cell.lineItemIds);
          divisionTotals[estId].lineItemCount += cell.lineItemCount;
        }

        // Add "Other" row for items on division code
        rows.push({
          code: division.code,
          label: 'Other',
          fullLabel: `${division.label} → Other`,
          level: 2,
          isSubtotal: false,
          amounts: this.mapAmountsToEstimates(divisionDirectAmounts, estimateIds),
          computed: this.computeStats(divisionDirectAmounts, estimateIds),
        });
      }
    }

    // Add subtotal row if division has data
    if (hasData) {
      rows.push({
        code: `${division.code}_SUBTOTAL`,
        label: division.label,
        fullLabel: `${division.label} Subtotal`,
        level: 1,
        isSubtotal: true,
        amounts: this.mapAmountsToEstimates(divisionTotals, estimateIds),
        computed: this.computeStats(divisionTotals, estimateIds),
      });
    }

    return rows;
  }

  /**
   * Build the grand total row
   */
  private buildGrandTotalRow(
    estimateIds: string[],
    codeAmounts: Map<string, Record<string, ComparisonCell>>
  ): ComparisonRow {
    const grandTotals: Record<string, ComparisonCell> = {};

    for (const amounts of codeAmounts.values()) {
      for (const [estId, cell] of Object.entries(amounts)) {
        if (!grandTotals[estId]) {
          grandTotals[estId] = {
            total: 0,
            lineItemIds: [],
            lineItemCount: 0,
          };
        }
        grandTotals[estId].total += cell.total;
        grandTotals[estId].lineItemIds.push(...cell.lineItemIds);
        grandTotals[estId].lineItemCount += cell.lineItemCount;
      }
    }

    return {
      code: 'GRAND_TOTAL',
      label: 'Grand Total',
      fullLabel: 'Grand Total',
      level: 0,
      isSubtotal: true,
      amounts: this.mapAmountsToEstimates(grandTotals, estimateIds),
      computed: this.computeStats(grandTotals, estimateIds),
    };
  }

  /**
   * Map amounts to all estimate IDs (including nulls for missing)
   */
  private mapAmountsToEstimates(
    amounts: Record<string, ComparisonCell>,
    estimateIds: string[]
  ): Record<string, ComparisonCell | null> {
    const result: Record<string, ComparisonCell | null> = {};

    for (const estId of estimateIds) {
      result[estId] = amounts[estId] || null;
    }

    return result;
  }

  /**
   * Compute statistics for a set of amounts
   */
  private computeStats(
    amounts: Record<string, ComparisonCell>,
    estimateIds: string[]
  ): ComputedStats {
    const values = estimateIds
      .map(id => amounts[id]?.total)
      .filter((v): v is number => v !== undefined && v !== null && v > 0);

    if (values.length === 0) {
      return {
        min: null,
        max: null,
        average: null,
        range: null,
        rangePercent: null,
      };
    }

    const min = Math.min(...values);
    const max = Math.max(...values);
    const sum = values.reduce((a, b) => a + b, 0);
    const average = sum / values.length;
    const range = max - min;
    const rangePercent = average > 0 ? range / average : null;

    return {
      min,
      max,
      average,
      range,
      rangePercent,
    };
  }

  /**
   * Calculate overall comparison summary
   */
  private calculateSummary(
    estimates: Estimate[],
    codeAmounts: Map<string, Record<string, ComparisonCell>>
  ): ComparisonSummary {
    const byEstimate: Record<string, number> = {};

    // Calculate total for each estimate
    for (const amounts of codeAmounts.values()) {
      for (const [estId, cell] of Object.entries(amounts)) {
        byEstimate[estId] = (byEstimate[estId] || 0) + cell.total;
      }
    }

    const totals = Object.values(byEstimate);

    if (totals.length === 0) {
      return {
        byEstimate: {},
        range: { min: 0, max: 0, spread: 0 },
        average: 0,
      };
    }

    const min = Math.min(...totals);
    const max = Math.max(...totals);

    return {
      byEstimate,
      range: {
        min,
        max,
        spread: max - min,
      },
      average: totals.reduce((a, b) => a + b, 0) / totals.length,
    };
  }

  /**
   * Create empty comparison for zero estimates
   */
  private createEmptyComparison(): Comparison {
    return {
      id: crypto.randomUUID(),
      sessionId: '',
      generatedAt: new Date().toISOString(),
      rows: [],
      summary: {
        byEstimate: {},
        range: { min: 0, max: 0, spread: 0 },
        average: 0,
      },
    };
  }
}
```

### Statistics Helpers (statistics.ts)

```typescript
/**
 * Calculate standard deviation
 */
export function standardDeviation(values: number[]): number {
  if (values.length === 0) return 0;
  
  const mean = values.reduce((a, b) => a + b, 0) / values.length;
  const squareDiffs = values.map(v => Math.pow(v - mean, 2));
  const avgSquareDiff = squareDiffs.reduce((a, b) => a + b, 0) / values.length;
  
  return Math.sqrt(avgSquareDiff);
}

/**
 * Calculate coefficient of variation (CV)
 */
export function coefficientOfVariation(values: number[]): number | null {
  if (values.length === 0) return null;
  
  const mean = values.reduce((a, b) => a + b, 0) / values.length;
  if (mean === 0) return null;
  
  const stdDev = standardDeviation(values);
  return stdDev / mean;
}

/**
 * Detect outliers using IQR method
 */
export function detectOutliers(
  values: number[],
  multiplier: number = 1.5
): { value: number; index: number; type: 'low' | 'high' }[] {
  if (values.length < 4) return [];
  
  const sorted = [...values].sort((a, b) => a - b);
  const q1Index = Math.floor(sorted.length * 0.25);
  const q3Index = Math.floor(sorted.length * 0.75);
  
  const q1 = sorted[q1Index];
  const q3 = sorted[q3Index];
  const iqr = q3 - q1;
  
  const lowerBound = q1 - multiplier * iqr;
  const upperBound = q3 + multiplier * iqr;
  
  const outliers: { value: number; index: number; type: 'low' | 'high' }[] = [];
  
  values.forEach((v, i) => {
    if (v < lowerBound) {
      outliers.push({ value: v, index: i, type: 'low' });
    } else if (v > upperBound) {
      outliers.push({ value: v, index: i, type: 'high' });
    }
  });
  
  return outliers;
}

/**
 * Calculate percentile
 */
export function percentile(values: number[], p: number): number | null {
  if (values.length === 0) return null;
  
  const sorted = [...values].sort((a, b) => a - b);
  const index = (p / 100) * (sorted.length - 1);
  const lower = Math.floor(index);
  const upper = Math.ceil(index);
  
  if (lower === upper) {
    return sorted[lower];
  }
  
  const weight = index - lower;
  return sorted[lower] * (1 - weight) + sorted[upper] * weight;
}
```

### Comparison Analysis Utilities

```typescript
import type { Comparison, ComparisonRow, Estimate } from '@estimate-compare/shared';

/**
 * Find codes that are missing from some estimates
 */
export function findMissingCodes(
  comparison: Comparison,
  estimates: Estimate[]
): Array<{
  code: string;
  label: string;
  missingFrom: string[];  // Estimate IDs
  presentIn: string[];    // Estimate IDs
}> {
  const missing: Array<{
    code: string;
    label: string;
    missingFrom: string[];
    presentIn: string[];
  }> = [];

  const estimateIds = estimates.map(e => e.id);

  for (const row of comparison.rows) {
    // Skip subtotal rows
    if (row.isSubtotal) continue;

    const presentIn: string[] = [];
    const missingFrom: string[] = [];

    for (const estId of estimateIds) {
      if (row.amounts[estId] && row.amounts[estId]!.total > 0) {
        presentIn.push(estId);
      } else {
        missingFrom.push(estId);
      }
    }

    // If not in all estimates, it's a potential scope difference
    if (missingFrom.length > 0 && presentIn.length > 0) {
      missing.push({
        code: row.code,
        label: row.fullLabel,
        missingFrom,
        presentIn,
      });
    }
  }

  return missing;
}

/**
 * Find significant price variances
 */
export function findSignificantVariances(
  comparison: Comparison,
  threshold: number = 0.3  // 30% variance threshold
): Array<{
  code: string;
  label: string;
  rangePercent: number;
  amounts: Record<string, number | null>;
}> {
  const variances: Array<{
    code: string;
    label: string;
    rangePercent: number;
    amounts: Record<string, number | null>;
  }> = [];

  for (const row of comparison.rows) {
    // Skip subtotal rows and rows without variance data
    if (row.isSubtotal) continue;
    if (row.computed.rangePercent === null) continue;

    if (row.computed.rangePercent > threshold) {
      const amounts: Record<string, number | null> = {};
      for (const [estId, cell] of Object.entries(row.amounts)) {
        amounts[estId] = cell?.total ?? null;
      }

      variances.push({
        code: row.code,
        label: row.fullLabel,
        rangePercent: row.computed.rangePercent,
        amounts,
      });
    }
  }

  // Sort by variance (highest first)
  return variances.sort((a, b) => b.rangePercent - a.rangePercent);
}

/**
 * Get comparison by division (for charts)
 */
export function getComparisonByDivision(
  comparison: Comparison
): Array<{
  division: string;
  amounts: Record<string, number>;
}> {
  const divisions: Array<{
    division: string;
    amounts: Record<string, number>;
  }> = [];

  for (const row of comparison.rows) {
    // Only look at division subtotals
    if (!row.isSubtotal || row.code === 'GRAND_TOTAL') continue;

    const amounts: Record<string, number> = {};
    for (const [estId, cell] of Object.entries(row.amounts)) {
      amounts[estId] = cell?.total ?? 0;
    }

    divisions.push({
      division: row.label,
      amounts,
    });
  }

  return divisions;
}
```

## Usage

```typescript
// In session processing

import { ComparisonGenerator } from '../services/comparison';
import { getCostCodesByFormat, getEstimatesBySession, saveComparison } from '../db/queries';

async function generateComparison(
  env: Env,
  sessionId: string
): Promise<Comparison> {
  // Load cost codes
  const session = await getSession(env.DB, sessionId);
  const costCodes = await getCostCodesByFormat(env.DB, session.formatId);
  
  // Load estimates
  const estimates = await getEstimatesBySession(env.DB, sessionId);
  
  // Generate comparison
  const generator = new ComparisonGenerator(costCodes);
  const comparison = generator.generate(estimates);
  
  // Save to database
  await saveComparison(env.DB, comparison);
  
  return comparison;
}

// After reclassification

async function regenerateComparison(
  env: Env,
  sessionId: string
): Promise<Comparison> {
  const session = await getSession(env.DB, sessionId);
  const costCodes = await getCostCodesByFormat(env.DB, session.formatId);
  const estimates = await getEstimatesBySession(env.DB, sessionId);
  
  const generator = new ComparisonGenerator(costCodes);
  const comparison = generator.regenerate(sessionId, estimates);
  
  await saveComparison(env.DB, comparison);
  
  return comparison;
}
```

## Todo List

### Core Generator

- [ ] Create comparison.ts file
- [ ] Implement ComparisonGenerator class
- [ ] Implement generate method
- [ ] Implement regenerate method
- [ ] Implement aggregateByCode method
- [ ] Implement buildRows method
- [ ] Implement buildDivisionRows method
- [ ] Implement buildGrandTotalRow method
- [ ] Implement mapAmountsToEstimates helper
- [ ] Implement computeStats method
- [ ] Implement calculateSummary method
- [ ] Implement createEmptyComparison method

### Statistics

- [ ] Create statistics.ts file
- [ ] Implement standardDeviation function
- [ ] Implement coefficientOfVariation function
- [ ] Implement detectOutliers function
- [ ] Implement percentile function

### Analysis Utilities

- [ ] Implement findMissingCodes function
- [ ] Implement findSignificantVariances function
- [ ] Implement getComparisonByDivision function

### Testing

- [ ] Test with 2 estimates
- [ ] Test with 3+ estimates
- [ ] Test with missing codes in some estimates
- [ ] Test with empty estimates
- [ ] Test subtotal calculation
- [ ] Test grand total calculation
- [ ] Test statistics (min, max, avg, spread)
- [ ] Test row ordering
- [ ] Test regeneration after reclassification

## Verification Checklist

- [ ] All line items are aggregated into comparison
- [ ] Division subtotals are correct
- [ ] Grand total matches estimate totals
- [ ] Statistics are calculated correctly
- [ ] Missing items are shown as null (not 0)
- [ ] Rows are in correct hierarchy order
- [ ] Unclassified items are handled
- [ ] Empty estimates don't break comparison
- [ ] Regeneration produces consistent results

## Notes

### Row Ordering

Rows should appear in this order:
1. Division 00 detail rows
2. Division 00 subtotal
3. Division 01 detail rows
4. Division 01 subtotal
5. ... (for all divisions with data)
6. Unclassified (if any)
7. Grand Total

### Handling Edge Cases

| Case | Behavior |
|------|----------|
| No estimates | Return empty comparison |
| One estimate | Return with null for stats that need multiple values |
| Empty estimate | Include with zeros |
| Unclassified items | Group in "Unclassified" row |
| Duplicate codes | Aggregate (should not happen) |

### Performance Considerations

- Comparison generation is O(n) where n = total line items
- No database queries during generation (data pre-loaded)
- Regeneration should be fast (<100ms for typical estimates)

### Sample Output

```json
{
  "id": "abc123",
  "sessionId": "session-1",
  "generatedAt": "2024-01-15T10:00:00Z",
  "rows": [
    {
      "code": "03_L",
      "label": "Labor",
      "fullLabel": "Framing → Labor",
      "level": 2,
      "isSubtotal": false,
      "amounts": {
        "est-1": { "total": 5000, "lineItemIds": ["item-1"], "lineItemCount": 1 },
        "est-2": { "total": 4500, "lineItemIds": ["item-2", "item-3"], "lineItemCount": 2 }
      },
      "computed": {
        "min": 4500,
        "max": 5000,
        "average": 4750,
        "range": 500,
        "rangePercent": 0.105
      }
    },
    {
      "code": "03_SUBTOTAL",
      "label": "Framing",
      "fullLabel": "Framing Subtotal",
      "level": 1,
      "isSubtotal": true,
      "amounts": { ... },
      "computed": { ... }
    }
  ],
  "summary": {
    "byEstimate": { "est-1": 50000, "est-2": 48000 },
    "range": { "min": 48000, "max": 50000, "spread": 2000 },
    "average": 49000
  }
}
```

## Time Estimate

| Task | Estimate |
|------|----------|
| Core generator | 2.5 hours |
| Statistics helpers | 45 min |
| Analysis utilities | 1 hour |
| Testing | 1.5 hours |
| **Total** | **~5.75 hours** |
