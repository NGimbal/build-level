# Module 12: Estimate Table

## Overview

Individual estimate detail table showing extracted line items with classification info. Includes the reclassification modal for manual code adjustments.

## Goals

- Display all line items from a single estimate
- Show extracted data (description, qty, unit, prices)
- Show classification (code, confidence, override status)
- Enable manual reclassification via modal
- Display estimate metadata (contractor, client, totals)
- Show notes and exclusions

## File Structure

```
packages/frontend/src/components/
├── EstimateTable.tsx         # Main estimate detail table
├── EstimateHeader.tsx        # Estimate metadata display
├── LineItemRow.tsx           # Individual line item row
├── ReclassifyModal.tsx       # Code selection modal
├── ConfidenceBadge.tsx       # Confidence score display
└── EstimateNotes.tsx         # Notes/exclusions section
```

## Core Implementation

### EstimateTable.tsx

```tsx
import { useState, useMemo } from 'preact/hooks';
import {
  useReactTable,
  getCoreRowModel,
  getSortedRowModel,
  flexRender,
  createColumnHelper,
  type SortingState,
} from '@tanstack/react-table';
import { useEstimate, useReclassifyLineItem, useCostCodes } from '../api/queries';
import type { LineItem, Estimate } from '@estimate-compare/shared';
import { formatCurrency } from '../lib/formatting';
import { EstimateHeader } from './EstimateHeader';
import { ReclassifyModal } from './ReclassifyModal';
import { ConfidenceBadge } from './ConfidenceBadge';
import { EstimateNotes } from './EstimateNotes';

interface EstimateTableProps {
  estimateId: string;
}

const columnHelper = createColumnHelper<LineItem>();

export function EstimateTable({ estimateId }: EstimateTableProps) {
  const { data: estimate, isLoading, error } = useEstimate(estimateId);
  const { data: costCodes } = useCostCodes();
  const reclassifyMutation = useReclassifyLineItem();
  
  const [editingItem, setEditingItem] = useState<LineItem | null>(null);
  const [sorting, setSorting] = useState<SortingState>([]);

  // Build code lookup map
  const codeMap = useMemo(() => {
    if (!costCodes) return new Map();
    return new Map(costCodes.map(c => [c.code, c]));
  }, [costCodes]);

  // Table columns
  const columns = useMemo(() => [
    // Line number
    columnHelper.accessor('lineNumber', {
      header: '#',
      size: 40,
      cell: ({ getValue }) => (
        <span class="line-number">{getValue() || '—'}</span>
      ),
    }),

    // Description with section header
    columnHelper.accessor('description', {
      header: 'Description',
      size: 350,
      cell: ({ row }) => {
        const { description, sectionHeader } = row.original;
        
        return (
          <div class="description-cell">
            {sectionHeader && (
              <span class="section-badge">{sectionHeader}</span>
            )}
            <span class="description-text">{description}</span>
          </div>
        );
      },
    }),

    // Quantity
    columnHelper.accessor('quantity', {
      header: 'Qty',
      size: 70,
      cell: ({ getValue }) => {
        const value = getValue();
        if (value === null || value === undefined) {
          return <span class="text-muted">—</span>;
        }
        return <span class="numeric">{value.toLocaleString()}</span>;
      },
    }),

    // Unit
    columnHelper.accessor('unit', {
      header: 'Unit',
      size: 60,
      cell: ({ getValue }) => (
        <span class="unit">{getValue() || '—'}</span>
      ),
    }),

    // Unit Price
    columnHelper.accessor('unitPrice', {
      header: 'Unit $',
      size: 90,
      cell: ({ getValue }) => {
        const value = getValue();
        if (value === null || value === undefined) {
          return <span class="text-muted">—</span>;
        }
        return <span class="numeric">{formatCurrency(value)}</span>;
      },
    }),

    // Total Price
    columnHelper.accessor('totalPrice', {
      header: 'Total',
      size: 100,
      cell: ({ getValue }) => (
        <span class="numeric font-medium">
          {formatCurrency(getValue())}
        </span>
      ),
    }),

    // Cost Code (clickable)
    columnHelper.accessor('costCode', {
      header: 'Code',
      size: 110,
      cell: ({ row }) => {
        const item = row.original;
        const code = item.costCode;
        const codeInfo = code ? codeMap.get(code) : null;
        
        return (
          <button
            class="code-button"
            onClick={() => setEditingItem(item)}
            title={codeInfo ? `${codeInfo.label}\nClick to reclassify` : 'Click to classify'}
          >
            <span class="code-value">{code || 'Unassigned'}</span>
            {item.isManualOverride && (
              <span class="override-badge" title="Manually classified">✎</span>
            )}
          </button>
        );
      },
    }),

    // Confidence
    columnHelper.accessor('confidence', {
      header: 'Conf.',
      size: 70,
      cell: ({ row }) => {
        const { confidence, isManualOverride } = row.original;
        
        if (isManualOverride) {
          return <span class="confidence-manual">Manual</span>;
        }
        
        return <ConfidenceBadge confidence={confidence} />;
      },
    }),
  ], [codeMap]);

  // Table instance
  const table = useReactTable({
    data: estimate?.lineItems || [],
    columns,
    state: { sorting },
    onSortingChange: setSorting,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
  });

  // Handle reclassification
  const handleReclassify = async (newCode: string) => {
    if (!editingItem) return;
    
    try {
      await reclassifyMutation.mutateAsync({
        estimateId,
        lineItemId: editingItem.id,
        newCode,
      });
      setEditingItem(null);
    } catch (error) {
      console.error('Reclassification failed:', error);
      // Error handled by mutation
    }
  };

  // Loading state
  if (isLoading) {
    return (
      <div class="estimate-loading">
        <div class="spinner" />
        <span>Loading estimate...</span>
      </div>
    );
  }

  // Error state
  if (error || !estimate) {
    return (
      <div class="estimate-error">
        <span>Failed to load estimate</span>
        <span class="error-detail">{error?.message}</span>
      </div>
    );
  }

  return (
    <div class="estimate-table-container">
      {/* Estimate header info */}
      <EstimateHeader estimate={estimate} />

      {/* Line items table */}
      <div class="table-wrapper">
        <table class="estimate-table">
          <thead>
            {table.getHeaderGroups().map((headerGroup) => (
              <tr key={headerGroup.id}>
                {headerGroup.headers.map((header) => (
                  <th
                    key={header.id}
                    style={{ width: header.getSize() }}
                    class={header.column.getCanSort() ? 'sortable' : ''}
                    onClick={header.column.getToggleSortingHandler()}
                  >
                    <div class="th-content">
                      {flexRender(
                        header.column.columnDef.header,
                        header.getContext()
                      )}
                      {header.column.getIsSorted() && (
                        <span class="sort-indicator">
                          {header.column.getIsSorted() === 'asc' ? '↑' : '↓'}
                        </span>
                      )}
                    </div>
                  </th>
                ))}
              </tr>
            ))}
          </thead>
          <tbody>
            {table.getRowModel().rows.map((row) => (
              <tr key={row.id} class="line-item-row">
                {row.getVisibleCells().map((cell) => (
                  <td key={cell.id}>
                    {flexRender(
                      cell.column.columnDef.cell,
                      cell.getContext()
                    )}
                  </td>
                ))}
              </tr>
            ))}
          </tbody>
          <tfoot>
            <tr class="totals-row">
              <td colSpan={5}>Total</td>
              <td class="numeric font-bold">
                {formatCurrency(estimate.documentTotals?.grandTotal)}
              </td>
              <td colSpan={2}></td>
            </tr>
          </tfoot>
        </table>
      </div>

      {/* Notes section */}
      {estimate.notes.length > 0 && (
        <EstimateNotes notes={estimate.notes} />
      )}

      {/* Reclassify modal */}
      {editingItem && costCodes && (
        <ReclassifyModal
          lineItem={editingItem}
          costCodes={costCodes}
          isLoading={reclassifyMutation.isPending}
          onSelect={handleReclassify}
          onClose={() => setEditingItem(null)}
        />
      )}
    </div>
  );
}
```

### EstimateHeader.tsx

```tsx
import type { Estimate } from '@estimate-compare/shared';
import { formatCurrency } from '../lib/formatting';

interface EstimateHeaderProps {
  estimate: Estimate;
}

export function EstimateHeader({ estimate }: EstimateHeaderProps) {
  const { contractor, client, documentTotals, filename } = estimate;

  return (
    <div class="estimate-header">
      <div class="header-main">
        <div class="contractor-info">
          <h2 class="contractor-name">
            {contractor?.name || filename}
          </h2>
          {contractor?.address && (
            <p class="contractor-address">{contractor.address}</p>
          )}
          <div class="contractor-contact">
            {contractor?.phone && <span>{contractor.phone}</span>}
            {contractor?.email && <span>{contractor.email}</span>}
          </div>
        </div>

        <div class="estimate-totals">
          <div class="total-row">
            <span class="total-label">Subtotal</span>
            <span class="total-value">
              {formatCurrency(documentTotals?.subtotal)}
            </span>
          </div>
          {documentTotals?.tax && (
            <div class="total-row">
              <span class="total-label">Tax</span>
              <span class="total-value">
                {formatCurrency(documentTotals.tax)}
              </span>
            </div>
          )}
          <div class="total-row grand-total">
            <span class="total-label">Total</span>
            <span class="total-value">
              {formatCurrency(documentTotals?.grandTotal)}
            </span>
          </div>
        </div>
      </div>

      {client && (
        <div class="client-info">
          <span class="client-label">Client:</span>
          <span class="client-name">{client.name}</span>
          {client.projectName && (
            <span class="project-name">• {client.projectName}</span>
          )}
        </div>
      )}

      <div class="item-count">
        {estimate.lineItems.length} line items
      </div>
    </div>
  );
}
```

### ReclassifyModal.tsx

```tsx
import { useState, useMemo, useRef, useEffect } from 'preact/hooks';
import type { LineItem, CostCode } from '@estimate-compare/shared';
import { formatCurrency } from '../lib/formatting';

interface ReclassifyModalProps {
  lineItem: LineItem;
  costCodes: CostCode[];
  isLoading: boolean;
  onSelect: (code: string) => void;
  onClose: () => void;
}

export function ReclassifyModal({
  lineItem,
  costCodes,
  isLoading,
  onSelect,
  onClose,
}: ReclassifyModalProps) {
  const [search, setSearch] = useState('');
  const [selectedCode, setSelectedCode] = useState<string | null>(
    lineItem.costCode
  );
  const searchRef = useRef<HTMLInputElement>(null);

  // Focus search on mount
  useEffect(() => {
    searchRef.current?.focus();
  }, []);

  // Close on escape
  useEffect(() => {
    const handleEscape = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose();
    };
    document.addEventListener('keydown', handleEscape);
    return () => document.removeEventListener('keydown', handleEscape);
  }, [onClose]);

  // Group codes by division (level 1)
  const groupedCodes = useMemo(() => {
    const divisions = costCodes.filter(c => c.level === 1);
    return divisions.map(div => ({
      division: div,
      children: costCodes.filter(c => c.parentCode === div.code),
    }));
  }, [costCodes]);

  // Filter by search
  const filteredGroups = useMemo(() => {
    if (!search.trim()) return groupedCodes;

    const searchLower = search.toLowerCase();
    
    return groupedCodes
      .map(group => ({
        division: group.division,
        children: group.children.filter(c =>
          c.label.toLowerCase().includes(searchLower) ||
          c.code.toLowerCase().includes(searchLower) ||
          group.division.label.toLowerCase().includes(searchLower)
        ),
      }))
      .filter(group =>
        group.children.length > 0 ||
        group.division.label.toLowerCase().includes(searchLower)
      );
  }, [groupedCodes, search]);

  const handleSubmit = () => {
    if (selectedCode) {
      onSelect(selectedCode);
    }
  };

  return (
    <div class="modal-overlay" onClick={onClose}>
      <div class="modal reclassify-modal" onClick={(e) => e.stopPropagation()}>
        {/* Header */}
        <div class="modal-header">
          <h3>Reclassify Line Item</h3>
          <button class="modal-close" onClick={onClose} disabled={isLoading}>
            ×
          </button>
        </div>

        {/* Line item preview */}
        <div class="modal-body">
          <div class="line-item-preview">
            <p class="preview-description">{lineItem.description}</p>
            <p class="preview-amount">{formatCurrency(lineItem.totalPrice)}</p>
            {lineItem.sectionHeader && (
              <p class="preview-section">Section: {lineItem.sectionHeader}</p>
            )}
          </div>

          {/* Current classification */}
          {lineItem.costCode && (
            <div class="current-classification">
              <span class="current-label">Current:</span>
              <span class="current-code">{lineItem.costCode}</span>
            </div>
          )}

          {/* Search input */}
          <input
            ref={searchRef}
            type="text"
            class="code-search"
            placeholder="Search cost codes..."
            value={search}
            onInput={(e) => setSearch((e.target as HTMLInputElement).value)}
          />

          {/* Suggested alternatives */}
          {lineItem.alternateCodes.length > 0 && !search && (
            <div class="alternates-section">
              <span class="alternates-label">Suggested alternatives:</span>
              <div class="alternates-list">
                {lineItem.alternateCodes.map((alt) => {
                  const codeInfo = costCodes.find(c => c.code === alt.code);
                  return (
                    <button
                      key={alt.code}
                      class={`alternate-chip ${selectedCode === alt.code ? 'selected' : ''}`}
                      onClick={() => setSelectedCode(alt.code)}
                    >
                      <span class="chip-code">{alt.code}</span>
                      <span class="chip-label">{codeInfo?.label}</span>
                      <span class="chip-confidence">
                        {Math.round(alt.confidence * 100)}%
                      </span>
                    </button>
                  );
                })}
              </div>
            </div>
          )}

          {/* Code list */}
          <div class="code-list">
            {filteredGroups.map((group) => (
              <div key={group.division.code} class="code-group">
                <div class="code-group-header">
                  <span class="group-code">{group.division.code}</span>
                  <span class="group-label">{group.division.label}</span>
                </div>
                
                <div class="code-group-items">
                  {group.children.map((code) => (
                    <button
                      key={code.code}
                      class={`code-item ${selectedCode === code.code ? 'selected' : ''}`}
                      onClick={() => setSelectedCode(code.code)}
                    >
                      <span class="item-code">{code.code}</span>
                      <span class="item-label">{code.label}</span>
                    </button>
                  ))}
                  
                  {/* Division-level option */}
                  <button
                    class={`code-item code-item-other ${selectedCode === group.division.code ? 'selected' : ''}`}
                    onClick={() => setSelectedCode(group.division.code)}
                  >
                    <span class="item-code">{group.division.code}</span>
                    <span class="item-label">Other {group.division.label}</span>
                  </button>
                </div>
              </div>
            ))}
          </div>
        </div>

        {/* Footer */}
        <div class="modal-footer">
          <button
            class="btn btn-secondary"
            onClick={onClose}
            disabled={isLoading}
          >
            Cancel
          </button>
          <button
            class="btn btn-primary"
            onClick={handleSubmit}
            disabled={!selectedCode || isLoading}
          >
            {isLoading ? 'Saving...' : 'Apply'}
          </button>
        </div>
      </div>
    </div>
  );
}
```

### ConfidenceBadge.tsx

```tsx
interface ConfidenceBadgeProps {
  confidence: number;
}

export function ConfidenceBadge({ confidence }: ConfidenceBadgeProps) {
  const percent = Math.round(confidence * 100);
  
  let level: 'high' | 'medium' | 'low';
  if (confidence >= 0.8) {
    level = 'high';
  } else if (confidence >= 0.5) {
    level = 'medium';
  } else {
    level = 'low';
  }

  return (
    <span class={`confidence-badge confidence-${level}`} title={`${percent}% confidence`}>
      {percent}%
    </span>
  );
}
```

### EstimateNotes.tsx

```tsx
import type { EstimateNote } from '@estimate-compare/shared';

interface EstimateNotesProps {
  notes: EstimateNote[];
}

export function EstimateNotes({ notes }: EstimateNotesProps) {
  // Group by type
  const grouped = notes.reduce((acc, note) => {
    if (!acc[note.noteType]) {
      acc[note.noteType] = [];
    }
    acc[note.noteType].push(note);
    return acc;
  }, {} as Record<string, EstimateNote[]>);

  const typeLabels: Record<string, string> = {
    note: 'Notes',
    exclusion: 'Exclusions',
    disclaimer: 'Disclaimers',
    term: 'Terms & Conditions',
  };

  const typeOrder = ['exclusion', 'note', 'disclaimer', 'term'];

  return (
    <div class="estimate-notes">
      {typeOrder.map((type) => {
        const items = grouped[type];
        if (!items || items.length === 0) return null;

        return (
          <div key={type} class={`notes-section notes-${type}`}>
            <h4 class="notes-title">{typeLabels[type]}</h4>
            <ul class="notes-list">
              {items.map((note) => (
                <li key={note.id} class="note-item">
                  {note.content}
                </li>
              ))}
            </ul>
          </div>
        );
      })}
    </div>
  );
}
```

## Styles

### estimate-table.css

```css
/* Container */
.estimate-table-container {
  display: flex;
  flex-direction: column;
  height: 100%;
  background: white;
  border-radius: 8px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

/* Header */
.estimate-header {
  padding: 20px;
  border-bottom: 1px solid #e2e8f0;
}

.header-main {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 24px;
}

.contractor-name {
  font-size: 20px;
  font-weight: 600;
  color: #1e293b;
  margin: 0 0 4px;
}

.contractor-address {
  color: #64748b;
  margin: 0 0 4px;
  font-size: 14px;
}

.contractor-contact {
  display: flex;
  gap: 16px;
  font-size: 13px;
  color: #64748b;
}

.estimate-totals {
  text-align: right;
}

.total-row {
  display: flex;
  justify-content: flex-end;
  gap: 24px;
  padding: 4px 0;
}

.total-label {
  color: #64748b;
  font-size: 14px;
}

.total-value {
  font-variant-numeric: tabular-nums;
  min-width: 100px;
}

.grand-total {
  border-top: 1px solid #e2e8f0;
  margin-top: 4px;
  padding-top: 8px;
}

.grand-total .total-label,
.grand-total .total-value {
  font-weight: 600;
  font-size: 16px;
  color: #1e293b;
}

.client-info {
  margin-top: 12px;
  padding-top: 12px;
  border-top: 1px solid #f1f5f9;
  font-size: 14px;
  color: #64748b;
}

.client-name {
  color: #1e293b;
  font-weight: 500;
}

.item-count {
  margin-top: 8px;
  font-size: 13px;
  color: #94a3b8;
}

/* Table */
.table-wrapper {
  flex: 1;
  overflow: auto;
}

.estimate-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 14px;
}

.estimate-table th {
  position: sticky;
  top: 0;
  background: #f8fafc;
  padding: 12px 8px;
  text-align: left;
  font-weight: 600;
  color: #475569;
  border-bottom: 2px solid #e2e8f0;
  white-space: nowrap;
}

.estimate-table th.sortable {
  cursor: pointer;
}

.estimate-table th.sortable:hover {
  background: #f1f5f9;
}

.th-content {
  display: flex;
  align-items: center;
  gap: 4px;
}

.sort-indicator {
  font-size: 12px;
  color: #3b82f6;
}

.estimate-table td {
  padding: 10px 8px;
  border-bottom: 1px solid #f1f5f9;
  vertical-align: middle;
}

.line-item-row:hover {
  background: #f8fafc;
}

/* Cell types */
.line-number {
  color: #94a3b8;
  font-size: 12px;
}

.description-cell {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.section-badge {
  display: inline-block;
  font-size: 10px;
  font-weight: 500;
  text-transform: uppercase;
  color: #64748b;
  background: #f1f5f9;
  padding: 2px 6px;
  border-radius: 3px;
  width: fit-content;
}

.description-text {
  color: #1e293b;
}

.numeric {
  font-variant-numeric: tabular-nums;
  text-align: right;
}

.unit {
  color: #64748b;
  font-size: 12px;
}

/* Code button */
.code-button {
  display: flex;
  align-items: center;
  gap: 4px;
  padding: 4px 8px;
  background: #f1f5f9;
  border: 1px solid #e2e8f0;
  border-radius: 4px;
  font-size: 12px;
  font-family: monospace;
  cursor: pointer;
  transition: all 0.15s;
}

.code-button:hover {
  background: #e2e8f0;
  border-color: #cbd5e1;
}

.code-value {
  color: #1e293b;
}

.override-badge {
  color: #3b82f6;
  font-size: 10px;
}

/* Confidence badges */
.confidence-badge {
  display: inline-block;
  padding: 2px 6px;
  border-radius: 4px;
  font-size: 11px;
  font-weight: 500;
}

.confidence-high {
  background: #f0fdf4;
  color: #16a34a;
}

.confidence-medium {
  background: #fefce8;
  color: #ca8a04;
}

.confidence-low {
  background: #fef2f2;
  color: #dc2626;
}

.confidence-manual {
  font-size: 11px;
  color: #3b82f6;
  font-style: italic;
}

/* Footer totals */
.totals-row {
  background: #f8fafc;
  font-weight: 600;
}

.totals-row td {
  padding: 16px 8px;
  border-top: 2px solid #e2e8f0;
}

/* Notes section */
.estimate-notes {
  padding: 20px;
  border-top: 1px solid #e2e8f0;
  background: #fafafa;
}

.notes-section {
  margin-bottom: 16px;
}

.notes-section:last-child {
  margin-bottom: 0;
}

.notes-title {
  font-size: 14px;
  font-weight: 600;
  color: #475569;
  margin: 0 0 8px;
}

.notes-exclusion .notes-title {
  color: #dc2626;
}

.notes-list {
  margin: 0;
  padding-left: 20px;
}

.note-item {
  color: #64748b;
  font-size: 13px;
  margin-bottom: 4px;
}

/* Modal styles */
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 100;
}

.reclassify-modal {
  width: 90%;
  max-width: 500px;
  max-height: 80vh;
  background: white;
  border-radius: 12px;
  display: flex;
  flex-direction: column;
  box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.25);
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 20px;
  border-bottom: 1px solid #e2e8f0;
}

.modal-header h3 {
  margin: 0;
  font-size: 18px;
}

.modal-close {
  background: none;
  border: none;
  font-size: 24px;
  color: #94a3b8;
  cursor: pointer;
  padding: 0;
  line-height: 1;
}

.modal-close:hover {
  color: #475569;
}

.modal-body {
  flex: 1;
  overflow: auto;
  padding: 20px;
}

.line-item-preview {
  background: #f8fafc;
  padding: 12px;
  border-radius: 6px;
  margin-bottom: 16px;
}

.preview-description {
  margin: 0 0 4px;
  font-weight: 500;
  color: #1e293b;
}

.preview-amount {
  margin: 0;
  color: #64748b;
}

.preview-section {
  margin: 4px 0 0;
  font-size: 12px;
  color: #94a3b8;
}

.code-search {
  width: 100%;
  padding: 10px 12px;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  font-size: 14px;
  margin-bottom: 16px;
}

.code-search:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.alternates-section {
  margin-bottom: 16px;
}

.alternates-label {
  display: block;
  font-size: 12px;
  color: #64748b;
  margin-bottom: 8px;
}

.alternates-list {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.alternate-chip {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 6px 10px;
  background: #f1f5f9;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.15s;
}

.alternate-chip:hover {
  background: #e2e8f0;
}

.alternate-chip.selected {
  background: #3b82f6;
  border-color: #3b82f6;
  color: white;
}

.chip-code {
  font-family: monospace;
  font-size: 11px;
}

.chip-label {
  font-size: 12px;
}

.chip-confidence {
  font-size: 10px;
  opacity: 0.7;
}

.code-list {
  max-height: 300px;
  overflow: auto;
}

.code-group {
  margin-bottom: 12px;
}

.code-group-header {
  display: flex;
  gap: 8px;
  padding: 8px;
  background: #f8fafc;
  border-radius: 4px;
  font-size: 13px;
  font-weight: 500;
  color: #475569;
}

.group-code {
  font-family: monospace;
}

.code-group-items {
  padding: 4px 0 4px 16px;
}

.code-item {
  display: flex;
  gap: 8px;
  width: 100%;
  padding: 8px;
  background: none;
  border: none;
  text-align: left;
  cursor: pointer;
  border-radius: 4px;
  transition: background 0.15s;
}

.code-item:hover {
  background: #f1f5f9;
}

.code-item.selected {
  background: #eff6ff;
}

.item-code {
  font-family: monospace;
  font-size: 12px;
  color: #64748b;
  min-width: 40px;
}

.item-label {
  font-size: 13px;
  color: #1e293b;
}

.code-item-other {
  border-top: 1px solid #f1f5f9;
  margin-top: 4px;
}

.modal-footer {
  display: flex;
  justify-content: flex-end;
  gap: 12px;
  padding: 16px 20px;
  border-top: 1px solid #e2e8f0;
}

/* Buttons */
.btn {
  padding: 8px 16px;
  border-radius: 6px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.15s;
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-primary {
  background: #3b82f6;
  color: white;
  border: none;
}

.btn-primary:hover:not(:disabled) {
  background: #2563eb;
}

.btn-secondary {
  background: white;
  color: #475569;
  border: 1px solid #e2e8f0;
}

.btn-secondary:hover:not(:disabled) {
  background: #f8fafc;
}

/* Loading/Error states */
.estimate-loading,
.estimate-error {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 48px;
  color: #64748b;
}

.spinner {
  width: 32px;
  height: 32px;
  border: 3px solid #e2e8f0;
  border-top-color: #3b82f6;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin-bottom: 16px;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.error-detail {
  font-size: 12px;
  color: #dc2626;
  margin-top: 8px;
}

/* Utilities */
.text-muted {
  color: #94a3b8;
}

.font-medium {
  font-weight: 500;
}

.font-bold {
  font-weight: 700;
}
```

## Todo List

### Core Table

- [ ] Create EstimateTable.tsx component
- [ ] Setup TanStack Table with columns
- [ ] Implement line number column
- [ ] Implement description column with section badge
- [ ] Implement quantity/unit/price columns
- [ ] Implement cost code column (clickable)
- [ ] Implement confidence column
- [ ] Add sorting capability
- [ ] Add totals footer row

### Header Component

- [ ] Create EstimateHeader.tsx
- [ ] Display contractor info
- [ ] Display client info
- [ ] Display document totals
- [ ] Show line item count

### Reclassify Modal

- [ ] Create ReclassifyModal.tsx
- [ ] Show line item preview
- [ ] Add search functionality
- [ ] Show suggested alternatives
- [ ] Group codes by division
- [ ] Handle selection state
- [ ] Handle loading state
- [ ] Handle escape key close

### Supporting Components

- [ ] Create ConfidenceBadge.tsx
- [ ] Create EstimateNotes.tsx
- [ ] Group notes by type

### Styling

- [ ] Create estimate-table.css
- [ ] Style header section
- [ ] Style table and cells
- [ ] Style code button
- [ ] Style confidence badges
- [ ] Style modal
- [ ] Style notes section

### Testing

- [ ] Test with estimate with all fields
- [ ] Test with sparse estimate data
- [ ] Test reclassification flow
- [ ] Test search filtering
- [ ] Test keyboard navigation
- [ ] Test loading/error states

## Verification Checklist

- [ ] All line items displayed correctly
- [ ] Section headers shown as badges
- [ ] Null values display as dashes
- [ ] Cost codes are clickable
- [ ] Override badge shows for manual items
- [ ] Confidence colors are correct
- [ ] Modal opens/closes properly
- [ ] Search filters codes correctly
- [ ] Alternatives are shown
- [ ] Reclassification updates immediately
- [ ] Notes grouped and displayed

## Time Estimate

| Task | Estimate |
|------|----------|
| Core table | 2 hours |
| Header component | 30 min |
| Reclassify modal | 2 hours |
| Supporting components | 45 min |
| Styling | 1.5 hours |
| Testing | 1 hour |
| **Total** | **~7.75 hours** |
