# Module 04: PDF Extraction

## Overview

PDF text extraction using pdfjs-serverless, optimized for construction estimate documents. This module handles reading PDF files from R2 storage and extracting structured text content.

## Goals

- Extract text from PDF documents uploaded to R2
- Preserve layout information (columns, tables)
- Handle multi-page documents
- Optimize for construction estimate formats
- Handle extraction errors gracefully

## File Structure

```
packages/backend/src/services/
├── pdf-extractor.ts      # Main extraction service
└── text-reconstruction.ts # Text layout reconstruction helpers
```

## Dependencies

```json
{
  "dependencies": {
    "pdfjs-serverless": "^0.4.0"
  }
}
```

## Core Implementation

### PDF Extractor Service (pdf-extractor.ts)

```typescript
import { getDocumentProxy } from 'pdfjs-serverless';

export interface ExtractedPage {
  pageNumber: number;
  text: string;
  width: number;
  height: number;
}

export interface PDFExtractionResult {
  pageCount: number;
  pages: ExtractedPage[];
  fullText: string;
  metadata?: {
    title?: string;
    author?: string;
    creationDate?: string;
  };
}

export interface ExtractionOptions {
  /** Maximum pages to extract (default: all) */
  maxPages?: number;
  /** Skip pages with no text content */
  skipEmptyPages?: boolean;
  /** Attempt to detect and preserve table structure */
  preserveTables?: boolean;
}

/**
 * Extract text content from a PDF document
 */
export async function extractPDFText(
  pdfBuffer: ArrayBuffer,
  options: ExtractionOptions = {}
): Promise<PDFExtractionResult> {
  const { 
    maxPages = Infinity, 
    skipEmptyPages = true,
    preserveTables = true,
  } = options;

  try {
    const pdf = await getDocumentProxy(new Uint8Array(pdfBuffer));
    const pages: ExtractedPage[] = [];
    
    const pageCount = Math.min(pdf.numPages, maxPages);

    for (let i = 1; i <= pageCount; i++) {
      const page = await pdf.getPage(i);
      const viewport = page.getViewport({ scale: 1.0 });
      const textContent = await page.getTextContent();
      
      const text = preserveTables
        ? reconstructTextWithLayout(textContent, viewport)
        : reconstructTextSimple(textContent);
      
      if (skipEmptyPages && !text.trim()) {
        continue;
      }

      pages.push({
        pageNumber: i,
        text,
        width: viewport.width,
        height: viewport.height,
      });
    }

    // Try to get document metadata
    let metadata: PDFExtractionResult['metadata'];
    try {
      const info = await pdf.getMetadata();
      metadata = {
        title: info?.info?.Title,
        author: info?.info?.Author,
        creationDate: info?.info?.CreationDate,
      };
    } catch {
      // Metadata extraction failed, continue without it
    }

    const fullText = pages
      .map((p, idx) => {
        if (pages.length > 1) {
          return `--- Page ${p.pageNumber} ---\n${p.text}`;
        }
        return p.text;
      })
      .join('\n\n');

    return {
      pageCount: pdf.numPages,
      pages,
      fullText,
      metadata,
    };

  } catch (error) {
    throw new PDFExtractionError(
      `Failed to extract PDF: ${error instanceof Error ? error.message : 'Unknown error'}`,
      error
    );
  }
}

export class PDFExtractionError extends Error {
  constructor(
    message: string,
    public readonly cause?: unknown
  ) {
    super(message);
    this.name = 'PDFExtractionError';
  }
}
```

### Text Reconstruction Helpers (text-reconstruction.ts)

```typescript
interface TextItem {
  str: string;
  transform: number[];  // [scaleX, skewX, skewY, scaleY, translateX, translateY]
  width: number;
  height: number;
  dir: string;
  fontName: string;
}

interface TextContent {
  items: TextItem[];
  styles: Record<string, unknown>;
}

interface Viewport {
  width: number;
  height: number;
}

interface TextLine {
  y: number;
  items: Array<{
    x: number;
    text: string;
    width: number;
    fontSize: number;
  }>;
}

/**
 * Simple text reconstruction - just concatenates text items
 */
export function reconstructTextSimple(textContent: TextContent): string {
  return textContent.items
    .map(item => item.str)
    .join(' ')
    .replace(/\s+/g, ' ')
    .trim();
}

/**
 * Layout-aware text reconstruction
 * Preserves columns and basic table structure
 */
export function reconstructTextWithLayout(
  textContent: TextContent,
  viewport: Viewport
): string {
  const items = textContent.items.filter(item => item.str.trim());
  
  if (items.length === 0) {
    return '';
  }

  // Group items into lines based on Y position
  const lines = groupIntoLines(items, viewport);
  
  // Sort lines top to bottom
  const sortedLines = Array.from(lines.values())
    .sort((a, b) => b.y - a.y);  // PDF Y is bottom-up
  
  // Build text output
  const textLines: string[] = [];
  
  for (const line of sortedLines) {
    const lineText = reconstructLine(line, viewport);
    textLines.push(lineText);
  }
  
  return textLines.join('\n');
}

/**
 * Group text items into lines based on Y position
 */
function groupIntoLines(items: TextItem[], viewport: Viewport): Map<number, TextLine> {
  const lines = new Map<number, TextLine>();
  const yTolerance = 3;  // Pixels tolerance for same line
  
  for (const item of items) {
    // Transform gives [scaleX, skewX, skewY, scaleY, translateX, translateY]
    const x = item.transform[4];
    const y = item.transform[5];
    const fontSize = Math.abs(item.transform[0]);  // Approximate font size from scale
    
    // Find or create line for this Y position
    let lineY: number | null = null;
    
    for (const existingY of lines.keys()) {
      if (Math.abs(existingY - y) <= yTolerance) {
        lineY = existingY;
        break;
      }
    }
    
    if (lineY === null) {
      lineY = y;
      lines.set(lineY, { y: lineY, items: [] });
    }
    
    lines.get(lineY)!.items.push({
      x,
      text: item.str,
      width: item.width,
      fontSize,
    });
  }
  
  return lines;
}

/**
 * Reconstruct a single line, preserving column spacing
 */
function reconstructLine(line: TextLine, viewport: Viewport): string {
  // Sort items left to right
  const sortedItems = line.items.sort((a, b) => a.x - b.x);
  
  if (sortedItems.length === 0) {
    return '';
  }
  
  let result = '';
  let lastX = 0;
  let lastWidth = 0;
  
  for (const item of sortedItems) {
    const gap = item.x - (lastX + lastWidth);
    
    // Determine spacing based on gap size
    if (result.length > 0) {
      if (gap > 50) {
        // Large gap - likely a column separator
        result += '\t\t';
      } else if (gap > 20) {
        // Medium gap - tab
        result += '\t';
      } else if (gap > 3) {
        // Small gap - space
        result += ' ';
      }
      // Very small gap - no separator (word continuation)
    }
    
    result += item.text;
    lastX = item.x;
    lastWidth = item.width;
  }
  
  return result;
}

/**
 * Detect if text appears to be in table format
 * Returns column boundaries if detected
 */
export function detectTableColumns(
  textContent: TextContent,
  viewport: Viewport
): number[] | null {
  const items = textContent.items.filter(item => item.str.trim());
  
  // Collect all X positions
  const xPositions: number[] = [];
  for (const item of items) {
    xPositions.push(item.transform[4]);
  }
  
  if (xPositions.length < 10) {
    return null;  // Not enough data to detect columns
  }
  
  // Find clusters of X positions (column alignments)
  const clusters = findClusters(xPositions, 10);  // 10px tolerance
  
  if (clusters.length >= 3 && clusters.length <= 10) {
    return clusters.sort((a, b) => a - b);
  }
  
  return null;
}

/**
 * Find clusters in a set of numbers
 */
function findClusters(values: number[], tolerance: number): number[] {
  if (values.length === 0) return [];
  
  const sorted = [...values].sort((a, b) => a - b);
  const clusters: { sum: number; count: number }[] = [];
  
  for (const value of sorted) {
    let added = false;
    
    for (const cluster of clusters) {
      const avg = cluster.sum / cluster.count;
      if (Math.abs(value - avg) <= tolerance) {
        cluster.sum += value;
        cluster.count++;
        added = true;
        break;
      }
    }
    
    if (!added) {
      clusters.push({ sum: value, count: 1 });
    }
  }
  
  // Return clusters with significant count (appear on multiple lines)
  return clusters
    .filter(c => c.count >= 3)
    .map(c => c.sum / c.count);
}
```

## R2 Integration

### Fetching PDF from R2

```typescript
import type { R2Bucket } from '@cloudflare/workers-types';
import { extractPDFText, PDFExtractionError } from './pdf-extractor';

export async function extractPDFFromR2(
  r2: R2Bucket,
  key: string
): Promise<PDFExtractionResult> {
  const object = await r2.get(key);
  
  if (!object) {
    throw new PDFExtractionError(`PDF not found in storage: ${key}`);
  }
  
  const buffer = await object.arrayBuffer();
  
  if (buffer.byteLength === 0) {
    throw new PDFExtractionError(`PDF file is empty: ${key}`);
  }
  
  // Basic PDF signature check
  const header = new Uint8Array(buffer.slice(0, 5));
  const signature = String.fromCharCode(...header);
  if (!signature.startsWith('%PDF')) {
    throw new PDFExtractionError(`File does not appear to be a valid PDF: ${key}`);
  }
  
  return extractPDFText(buffer);
}
```

## Construction Estimate Optimizations

### Estimate-Specific Extraction

```typescript
/**
 * Enhanced extraction for construction estimates
 * Attempts to identify and preserve common estimate structures
 */
export async function extractConstructionEstimate(
  pdfBuffer: ArrayBuffer
): Promise<{
  extraction: PDFExtractionResult;
  hints: EstimateHints;
}> {
  const extraction = await extractPDFText(pdfBuffer, {
    preserveTables: true,
  });
  
  // Analyze the extracted text for estimate patterns
  const hints = analyzeForEstimatePatterns(extraction.fullText);
  
  return { extraction, hints };
}

export interface EstimateHints {
  /** Likely has line item table */
  hasLineItemTable: boolean;
  /** Detected currency format */
  currencyFormat: 'USD' | 'unknown';
  /** Likely column positions */
  probableColumns: string[];
  /** Header area (first N lines likely header) */
  headerLineCount: number;
  /** Footer area (last N lines likely footer/notes) */
  footerLineCount: number;
}

function analyzeForEstimatePatterns(text: string): EstimateHints {
  const lines = text.split('\n');
  
  // Look for common estimate patterns
  const hints: EstimateHints = {
    hasLineItemTable: false,
    currencyFormat: 'unknown',
    probableColumns: [],
    headerLineCount: 0,
    footerLineCount: 0,
  };
  
  // Check for currency patterns
  if (/\$[\d,]+\.?\d*/g.test(text)) {
    hints.currencyFormat = 'USD';
  }
  
  // Check for table-like patterns (multiple amounts on many lines)
  const linesWithAmounts = lines.filter(line => 
    /\$?[\d,]+\.\d{2}/.test(line)
  );
  hints.hasLineItemTable = linesWithAmounts.length >= 5;
  
  // Detect common column headers
  const columnPatterns = [
    /description/i,
    /qty|quantity/i,
    /unit/i,
    /price|rate|cost/i,
    /amount|total|ext/i,
  ];
  
  for (const line of lines.slice(0, 20)) {
    const matchedPatterns = columnPatterns.filter(p => p.test(line));
    if (matchedPatterns.length >= 3) {
      hints.probableColumns = ['Description', 'Qty', 'Unit', 'Price', 'Total'];
      break;
    }
  }
  
  // Estimate header/footer sizes
  // Look for first line with currency amount
  for (let i = 0; i < lines.length; i++) {
    if (/\$?[\d,]+\.\d{2}/.test(lines[i])) {
      hints.headerLineCount = Math.max(0, i - 1);
      break;
    }
  }
  
  // Look for "total", "notes", "exclusions" near end
  for (let i = lines.length - 1; i >= 0; i--) {
    if (/total|subtotal|grand total/i.test(lines[i])) {
      hints.footerLineCount = lines.length - i - 1;
      break;
    }
  }
  
  return hints;
}
```

## Error Handling

### Common PDF Issues

```typescript
export type PDFIssue = 
  | 'encrypted'
  | 'corrupted'
  | 'empty'
  | 'scanned_image'
  | 'unsupported_format'
  | 'too_large';

export function diagnosePDFIssue(error: unknown, buffer?: ArrayBuffer): PDFIssue | null {
  const errorMessage = error instanceof Error ? error.message.toLowerCase() : '';
  
  if (errorMessage.includes('encrypted') || errorMessage.includes('password')) {
    return 'encrypted';
  }
  
  if (errorMessage.includes('invalid') || errorMessage.includes('corrupt')) {
    return 'corrupted';
  }
  
  if (buffer && buffer.byteLength === 0) {
    return 'empty';
  }
  
  if (buffer && buffer.byteLength > 50 * 1024 * 1024) {  // 50MB
    return 'too_large';
  }
  
  return null;
}

export function getPDFIssueMessage(issue: PDFIssue): string {
  const messages: Record<PDFIssue, string> = {
    encrypted: 'This PDF is password protected. Please upload an unprotected version.',
    corrupted: 'This PDF appears to be corrupted. Please try re-exporting the document.',
    empty: 'This PDF file is empty.',
    scanned_image: 'This PDF appears to be a scanned image. Text extraction is not available.',
    unsupported_format: 'This PDF format is not supported.',
    too_large: 'This PDF is too large. Maximum size is 50MB.',
  };
  
  return messages[issue];
}
```

## Usage in Processing Pipeline

```typescript
// In the session processing worker

import { extractPDFFromR2 } from '../services/pdf-extractor';
import { EstimateParser } from '../services/estimate-parser';

async function processEstimate(
  env: Env,
  estimateId: string,
  r2Key: string
): Promise<void> {
  // Update status
  await updateEstimate(env.DB, estimateId, { status: 'extracting' });
  
  try {
    // Extract text from PDF
    const extraction = await extractPDFFromR2(env.R2, r2Key);
    
    // Store raw text for debugging
    await updateEstimate(env.DB, estimateId, {
      rawText: extraction.fullText.slice(0, 50000),  // Limit stored text
    });
    
    // Continue to parsing...
    await updateEstimate(env.DB, estimateId, { status: 'classifying' });
    
    // Parse with LLM
    const parser = new EstimateParser(env.AI);
    const parsed = await parser.parse(extraction.fullText, filename);
    
    // ... rest of processing
    
  } catch (error) {
    const issue = diagnosePDFIssue(error);
    const message = issue 
      ? getPDFIssueMessage(issue)
      : `Extraction failed: ${error instanceof Error ? error.message : 'Unknown error'}`;
    
    await updateEstimate(env.DB, estimateId, {
      status: 'error',
      errorMessage: message,
    });
  }
}
```

## Todo List

### Core Extraction

- [ ] Install pdfjs-serverless dependency
- [ ] Create pdf-extractor.ts file
- [ ] Implement extractPDFText function
- [ ] Implement PDFExtractionError class
- [ ] Handle multi-page documents
- [ ] Extract document metadata

### Text Reconstruction

- [ ] Create text-reconstruction.ts file
- [ ] Implement reconstructTextSimple function
- [ ] Implement reconstructTextWithLayout function
- [ ] Implement groupIntoLines function
- [ ] Implement reconstructLine with column detection
- [ ] Implement detectTableColumns function
- [ ] Implement findClusters helper

### R2 Integration

- [ ] Implement extractPDFFromR2 function
- [ ] Add PDF signature validation
- [ ] Handle missing files gracefully
- [ ] Handle empty files

### Construction Estimate Optimizations

- [ ] Implement extractConstructionEstimate function
- [ ] Implement analyzeForEstimatePatterns function
- [ ] Detect currency format
- [ ] Detect column headers
- [ ] Estimate header/footer regions

### Error Handling

- [ ] Implement diagnosePDFIssue function
- [ ] Implement getPDFIssueMessage function
- [ ] Handle encrypted PDFs
- [ ] Handle corrupted PDFs
- [ ] Handle scanned PDFs (future: OCR)
- [ ] Handle oversized PDFs

### Testing

- [ ] Test with simple single-page PDF
- [ ] Test with multi-page estimate
- [ ] Test with columnar layout
- [ ] Test with table-heavy estimates
- [ ] Test error cases (encrypted, corrupted)
- [ ] Test R2 integration
- [ ] Verify text quality is sufficient for LLM parsing

## Verification Checklist

- [ ] pdfjs-serverless works in Cloudflare Workers
- [ ] Text extraction produces readable output
- [ ] Layout preservation maintains table structure
- [ ] Multi-page documents handled correctly
- [ ] Error messages are user-friendly
- [ ] Memory usage is acceptable for large PDFs
- [ ] Integration with R2 storage works

## Notes

### pdfjs-serverless Limitations

- No canvas rendering (text only)
- May have issues with some font encodings
- Cannot extract from scanned/image PDFs
- Memory limits in Workers environment

### Performance Considerations

```typescript
// For very large PDFs, consider streaming or chunking
const MAX_PDF_SIZE = 20 * 1024 * 1024;  // 20MB recommended max

if (buffer.byteLength > MAX_PDF_SIZE) {
  // Consider processing only first N pages
  return extractPDFText(buffer, { maxPages: 10 });
}
```

### Quality Improvement Ideas

1. **Font mapping**: Some PDFs use custom encodings
2. **OCR fallback**: For scanned documents (requires external service)
3. **Table detection**: Use heuristics to identify table boundaries
4. **Confidence scoring**: Rate extraction quality

### Sample Extracted Text

Good extraction should look like:

```
ABC Construction Company
123 Main Street, Anytown, USA 12345
Phone: (555) 123-4567

ESTIMATE

Customer: John Smith
Project: Kitchen Renovation
Date: January 15, 2024

Description                     Qty    Unit    Unit Price    Total
-----------------------------------------------------------------
Demo existing cabinets          1      LS      $500.00       $500.00
Install base cabinets          12      LF      $150.00     $1,800.00
Install upper cabinets          8      LF      $125.00     $1,000.00
Countertop - Granite           25      SF       $85.00     $2,125.00

                                              Subtotal:    $5,425.00
                                              Tax (8%):      $434.00
                                              TOTAL:       $5,859.00
```

## Time Estimate

| Task | Estimate |
|------|----------|
| Core extraction | 1.5 hours |
| Text reconstruction | 2 hours |
| R2 integration | 30 min |
| Estimate optimizations | 1 hour |
| Error handling | 30 min |
| Testing | 1 hour |
| **Total** | **~6.5 hours** |
