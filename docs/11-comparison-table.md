# Module 11: Comparison Table

## Overview

The main comparison table component that displays aggregated estimate data side-by-side with computed statistics. Built with TanStack Table for powerful data grid functionality.

## Goals

- Display comparison rows with hierarchical structure
- Show dynamic columns for each estimate
- Calculate and display statistics (min, max, avg, spread)
- Support visual highlighting for outliers
- Enable drill-down to line item details
- Handle responsive layout

## File Structure

```
packages/frontend/src/components/
├── ComparisonTable.tsx       # Main comparison table
├── ComparisonTableRow.tsx    # Individual row component
├── ComparisonStats.tsx       # Statistics display
├── ComparisonSummary.tsx     # Summary footer
└── ComparisonCharts.tsx      # Optional chart visualizations
```

## Core Implementation

### ComparisonTable.tsx

```tsx
import { useMemo, useState } from 'preact/hooks';
import {
  useReactTable,
  getCoreRowModel,
  getExpandedRowModel,
  flexRender,
  createColumnHelper,
  type ColumnDef,
  type Row,
} from '@tanstack/react-table';
import type { 
  Comparison, 
  ComparisonRow, 
  Estimate 
} from '@estimate-compare/shared';
import { formatCurrency, formatPercent } from '../lib/formatting';
import { ComparisonSummary } from './ComparisonSummary';

interface ComparisonTableProps {
  comparison: Comparison;
  estimates: Estimate[];
  onRowClick?: (row: ComparisonRow) => void;
}

const columnHelper = createColumnHelper<ComparisonRow>();

export function ComparisonTable({ 
  comparison, 
  estimates,
  onRowClick,
}: ComparisonTableProps) {
  const [hoveredRow, setHoveredRow] = useState<string | null>(null);

  // Build columns dynamically based on estimates
  const columns = useMemo(() => {
    const cols: ColumnDef<ComparisonRow, any>[] = [
      // Code column
      columnHelper.accessor('code', {
        header: 'Code',
        size: 80,
        cell: ({ row }) => {
          const data = row.original;
          
          // Hide code for subtotal/total rows
          if (data.isSubtotal || data.code === 'GRAND_TOTAL') {
            return null;
          }
          
          return (
            <span 
              class="code-cell"
              style={{ paddingLeft: `${(data.level - 1) * 16}px` }}
            >
              {data.code}
            </span>
          );
        },
      }),

      // Description column
      columnHelper.accessor('fullLabel', {
        header: 'Description',
        size: 220,
        cell: ({ row }) => {
          const data = row.original;
          const isTotal = data.isSubtotal || data.code === 'GRAND_TOTAL';
          const isGrandTotal = data.code === 'GRAND_TOTAL';
          
          return (
            <span
              class={`description-cell ${isTotal ? 'font-semibold' : ''} ${isGrandTotal ? 'text-lg' : ''}`}
              style={{ 
                paddingLeft: isTotal ? 0 : `${(data.level - 1) * 16}px` 
              }}
            >
              {data.fullLabel}
            </span>
          );
        },
      }),

      // Dynamic estimate columns
      ...estimates.map((estimate) =>
        columnHelper.accessor(
          (row) => row.amounts[estimate.id]?.total ?? null,
          {
            id: `estimate_${estimate.id}`,
            header: () => (
              <div class="estimate-header">
                <span class="estimate-name">
                  {estimate.contractor?.name || estimate.filename}
                </span>
                <span class="estimate-total">
                  {formatCurrency(estimate.documentTotals?.grandTotal)}
                </span>
              </div>
            ),
            size: 130,
            cell: ({ getValue, row }) => {
              const value = getValue();
              const data = row.original;
              const isTotal = data.isSubtotal || data.code === 'GRAND_TOTAL';
              
              if (value === null || value === undefined) {
                return <span class="text-muted">—</span>;
              }
              
              // Highlight cell based on comparison
              const { min, max, average } = data.computed;
              let highlight = '';
              
              if (min !== null && max !== null && min !== max) {
                if (value === min) highlight = 'cell-low';
                else if (value === max) highlight = 'cell-high';
              }
              
              return (
                <span class={`amount-cell ${highlight} ${isTotal ? 'font-semibold' : ''}`}>
                  {formatCurrency(value)}
                </span>
              );
            },
          }
        )
      ),

      // Divider
      columnHelper.display({
        id: 'divider',
        header: '',
        size: 2,
        cell: () => <div class="column-divider" />,
      }),

      // Statistics columns
      columnHelper.accessor('computed.min', {
        header: 'Min',
        size: 100,
        cell: ({ getValue, row }) => {
          const value = getValue();
          const isTotal = row.original.isSubtotal;
          
          return (
            <span class={`stat-cell stat-min ${isTotal ? 'font-semibold' : ''}`}>
              {value !== null ? formatCurrency(value) : '—'}
            </span>
          );
        },
      }),

      columnHelper.accessor('computed.max', {
        header: 'Max',
        size: 100,
        cell: ({ getValue, row }) => {
          const value = getValue();
          const isTotal = row.original.isSubtotal;
          
          return (
            <span class={`stat-cell stat-max ${isTotal ? 'font-semibold' : ''}`}>
              {value !== null ? formatCurrency(value) : '—'}
            </span>
          );
        },
      }),

      columnHelper.accessor('computed.average', {
        header: 'Average',
        size: 100,
        cell: ({ getValue, row }) => {
          const value = getValue();
          const isTotal = row.original.isSubtotal;
          
          return (
            <span class={`stat-cell ${isTotal ? 'font-semibold' : ''}`}>
              {value !== null ? formatCurrency(value) : '—'}
            </span>
          );
        },
      }),

      columnHelper.accessor('computed.rangePercent', {
        header: 'Spread',
        size: 80,
        cell: ({ getValue, row }) => {
          const value = getValue();
          
          if (value === null || value === undefined) {
            return <span class="text-muted">—</span>;
          }
          
          // Color code based on variance
          let colorClass = 'spread-low';
          if (value > 0.30) {
            colorClass = 'spread-high';
          } else if (value > 0.15) {
            colorClass = 'spread-medium';
          }
          
          return (
            <span class={`spread-cell ${colorClass}`}>
              {formatPercent(value)}
            </span>
          );
        },
      }),
    ];

    return cols;
  }, [estimates]);

  // Setup table instance
  const table = useReactTable({
    data: comparison.rows,
    columns,
    getCoreRowModel: getCoreRowModel(),
    getRowId: (row) => row.code,
  });

  return (
    <div class="comparison-table-container">
      <div class="table-scroll-wrapper">
        <table class="comparison-table">
          <thead>
            {table.getHeaderGroups().map((headerGroup) => (
              <tr key={headerGroup.id}>
                {headerGroup.headers.map((header) => (
                  <th
                    key={header.id}
                    style={{ width: header.getSize() }}
                    class={header.id.startsWith('estimate_') ? 'estimate-col' : ''}
                  >
                    {header.isPlaceholder
                      ? null
                      : flexRender(
                          header.column.columnDef.header,
                          header.getContext()
                        )}
                  </th>
                ))}
              </tr>
            ))}
          </thead>
          <tbody>
            {table.getRowModel().rows.map((row) => (
              <ComparisonTableRow
                key={row.id}
                row={row}
                isHovered={hoveredRow === row.id}
                onHover={setHoveredRow}
                onClick={onRowClick}
              />
            ))}
          </tbody>
        </table>
      </div>

      {/* Summary footer */}
      <ComparisonSummary 
        summary={comparison.summary} 
        estimates={estimates} 
      />
    </div>
  );
}
```

### ComparisonTableRow.tsx

```tsx
import { flexRender, type Row } from '@tanstack/react-table';
import type { ComparisonRow } from '@estimate-compare/shared';

interface ComparisonTableRowProps {
  row: Row<ComparisonRow>;
  isHovered: boolean;
  onHover: (id: string | null) => void;
  onClick?: (row: ComparisonRow) => void;
}

export function ComparisonTableRow({
  row,
  isHovered,
  onHover,
  onClick,
}: ComparisonTableRowProps) {
  const data = row.original;
  const isSubtotal = data.isSubtotal;
  const isGrandTotal = data.code === 'GRAND_TOTAL';
  const isClickable = !isSubtotal && onClick;

  const rowClasses = [
    'comparison-row',
    isGrandTotal && 'grand-total-row',
    isSubtotal && !isGrandTotal && 'subtotal-row',
    !isSubtotal && `level-${data.level}`,
    isHovered && 'hovered',
    isClickable && 'clickable',
  ].filter(Boolean).join(' ');

  return (
    <tr
      class={rowClasses}
      onMouseEnter={() => onHover(row.id)}
      onMouseLeave={() => onHover(null)}
      onClick={() => isClickable && onClick(data)}
    >
      {row.getVisibleCells().map((cell) => (
        <td key={cell.id}>
          {flexRender(cell.column.columnDef.cell, cell.getContext())}
        </td>
      ))}
    </tr>
  );
}
```

### ComparisonSummary.tsx

```tsx
import type { ComparisonSummary as SummaryType, Estimate } from '@estimate-compare/shared';
import { formatCurrency, formatPercent } from '../lib/formatting';

interface ComparisonSummaryProps {
  summary: SummaryType;
  estimates: Estimate[];
}

export function ComparisonSummary({ summary, estimates }: ComparisonSummaryProps) {
  // Find lowest and highest bidders
  const estimateTotals = estimates.map(e => ({
    id: e.id,
    name: e.contractor?.name || e.filename,
    total: summary.byEstimate[e.id] || 0,
  }));

  const sorted = [...estimateTotals].sort((a, b) => a.total - b.total);
  const lowest = sorted[0];
  const highest = sorted[sorted.length - 1];
  const spreadPercent = summary.average > 0 
    ? summary.range.spread / summary.average 
    : 0;

  return (
    <div class="comparison-summary">
      <div class="summary-card summary-lowest">
        <span class="summary-label">Lowest Bid</span>
        <span class="summary-value">{formatCurrency(lowest?.total)}</span>
        <span class="summary-name">{lowest?.name}</span>
      </div>

      <div class="summary-card summary-highest">
        <span class="summary-label">Highest Bid</span>
        <span class="summary-value">{formatCurrency(highest?.total)}</span>
        <span class="summary-name">{highest?.name}</span>
      </div>

      <div class="summary-card">
        <span class="summary-label">Spread</span>
        <span class="summary-value">{formatCurrency(summary.range.spread)}</span>
        <span class="summary-detail">{formatPercent(spreadPercent)}</span>
      </div>

      <div class="summary-card">
        <span class="summary-label">Average</span>
        <span class="summary-value">{formatCurrency(summary.average)}</span>
        <span class="summary-detail">{estimates.length} estimates</span>
      </div>
    </div>
  );
}
```

## Styles

### comparison-table.css

```css
/* Container */
.comparison-table-container {
  display: flex;
  flex-direction: column;
  height: 100%;
  background: white;
  border-radius: 8px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.table-scroll-wrapper {
  flex: 1;
  overflow: auto;
}

/* Table */
.comparison-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 14px;
}

/* Header */
.comparison-table thead {
  position: sticky;
  top: 0;
  z-index: 10;
  background: #f8fafc;
}

.comparison-table th {
  padding: 12px 8px;
  text-align: left;
  font-weight: 600;
  color: #475569;
  border-bottom: 2px solid #e2e8f0;
  white-space: nowrap;
}

.comparison-table th.estimate-col {
  text-align: right;
  background: #f1f5f9;
}

.estimate-header {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 2px;
}

.estimate-name {
  font-size: 13px;
  font-weight: 600;
  color: #1e293b;
  max-width: 120px;
  overflow: hidden;
  text-overflow: ellipsis;
}

.estimate-total {
  font-size: 11px;
  color: #64748b;
}

/* Cells */
.comparison-table td {
  padding: 10px 8px;
  border-bottom: 1px solid #f1f5f9;
}

.code-cell {
  font-family: monospace;
  font-size: 12px;
  color: #64748b;
}

.description-cell {
  color: #1e293b;
}

.amount-cell {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

.stat-cell {
  text-align: right;
  font-variant-numeric: tabular-nums;
  color: #64748b;
}

.stat-min {
  color: #16a34a;
}

.stat-max {
  color: #dc2626;
}

/* Spread coloring */
.spread-cell {
  text-align: right;
  font-weight: 500;
  padding: 2px 6px;
  border-radius: 4px;
}

.spread-low {
  color: #16a34a;
  background: #f0fdf4;
}

.spread-medium {
  color: #ca8a04;
  background: #fefce8;
}

.spread-high {
  color: #dc2626;
  background: #fef2f2;
}

/* Cell highlighting */
.cell-low {
  background: #f0fdf4;
  color: #16a34a;
  font-weight: 500;
}

.cell-high {
  background: #fef2f2;
  color: #dc2626;
  font-weight: 500;
}

/* Row types */
.comparison-row {
  transition: background-color 0.15s;
}

.comparison-row.hovered {
  background: #f8fafc;
}

.comparison-row.clickable {
  cursor: pointer;
}

.comparison-row.clickable:hover {
  background: #f1f5f9;
}

.subtotal-row {
  background: #f8fafc;
  border-top: 1px solid #e2e8f0;
}

.subtotal-row td {
  padding-top: 12px;
  padding-bottom: 12px;
}

.grand-total-row {
  background: #1e293b;
  color: white;
}

.grand-total-row td {
  padding: 16px 8px;
  border: none;
}

.grand-total-row .stat-cell,
.grand-total-row .amount-cell,
.grand-total-row .spread-cell {
  color: white;
}

.grand-total-row .spread-cell {
  background: rgba(255, 255, 255, 0.1);
}

/* Column divider */
.column-divider {
  width: 2px;
  background: #e2e8f0;
  height: 100%;
}

/* Summary footer */
.comparison-summary {
  display: flex;
  gap: 16px;
  padding: 16px;
  background: #f8fafc;
  border-top: 1px solid #e2e8f0;
}

.summary-card {
  flex: 1;
  display: flex;
  flex-direction: column;
  padding: 12px 16px;
  background: white;
  border-radius: 6px;
  border: 1px solid #e2e8f0;
}

.summary-lowest {
  border-left: 3px solid #16a34a;
}

.summary-highest {
  border-left: 3px solid #dc2626;
}

.summary-label {
  font-size: 12px;
  color: #64748b;
  margin-bottom: 4px;
}

.summary-value {
  font-size: 18px;
  font-weight: 600;
  color: #1e293b;
  font-variant-numeric: tabular-nums;
}

.summary-name,
.summary-detail {
  font-size: 12px;
  color: #94a3b8;
  margin-top: 2px;
}

/* Responsive */
@media (max-width: 768px) {
  .comparison-summary {
    flex-wrap: wrap;
  }
  
  .summary-card {
    flex: 1 1 calc(50% - 8px);
    min-width: 140px;
  }
}

/* Utilities */
.text-muted {
  color: #94a3b8;
}

.font-semibold {
  font-weight: 600;
}

.text-lg {
  font-size: 16px;
}
```

## Todo List

### Core Table

- [ ] Create ComparisonTable.tsx component
- [ ] Setup TanStack Table with dynamic columns
- [ ] Implement code column with hierarchy indentation
- [ ] Implement description column
- [ ] Implement dynamic estimate columns
- [ ] Implement statistics columns (min, max, avg, spread)
- [ ] Add column divider between estimates and stats

### Row Component

- [ ] Create ComparisonTableRow.tsx component
- [ ] Implement row type styling (normal, subtotal, grand total)
- [ ] Implement hover state
- [ ] Implement click handler for drill-down
- [ ] Add proper accessibility attributes

### Summary Component

- [ ] Create ComparisonSummary.tsx component
- [ ] Display lowest bid with contractor name
- [ ] Display highest bid with contractor name
- [ ] Display spread (absolute and percentage)
- [ ] Display average

### Styling

- [ ] Create comparison-table.css
- [ ] Style table header (sticky)
- [ ] Style row levels and types
- [ ] Style cell highlighting (min/max)
- [ ] Style spread color coding
- [ ] Style summary cards
- [ ] Add responsive breakpoints

### Features

- [ ] Add row hover effects
- [ ] Add click to expand/drill-down
- [ ] Add sort by column (optional)
- [ ] Add column resize (optional)
- [ ] Handle missing values display

### Testing

- [ ] Test with 2 estimates
- [ ] Test with 3+ estimates
- [ ] Test with missing data in some estimates
- [ ] Test spread calculations display
- [ ] Test summary accuracy
- [ ] Test responsive behavior

## Verification Checklist

- [ ] All estimates display as columns
- [ ] Statistics calculated correctly
- [ ] Row hierarchy displays properly
- [ ] Subtotal rows are visually distinct
- [ ] Grand total row is prominent
- [ ] Low/high values highlighted
- [ ] Spread percentage colored appropriately
- [ ] Summary shows correct lowest/highest
- [ ] Table scrolls properly with sticky header

## Notes

### TanStack Table Setup for Preact

```typescript
// May need React aliasing in vite.config.ts
resolve: {
  alias: {
    'react': 'preact/compat',
    'react-dom': 'preact/compat',
  },
},
```

### Dynamic Column Generation

The number of estimate columns is dynamic. Key considerations:
- Column IDs must be unique
- Header can be a component (shows contractor name + total)
- Cell accessor uses estimate ID to look up amounts

### Performance

For large comparisons (many rows):
- Use virtualization if >100 rows
- Memoize column definitions
- Consider pagination

## Time Estimate

| Task | Estimate |
|------|----------|
| Core table | 2 hours |
| Row component | 45 min |
| Summary component | 45 min |
| Styling | 1.5 hours |
| Features | 1 hour |
| Testing | 45 min |
| **Total** | **~6.75 hours** |
