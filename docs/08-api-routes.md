# Module 08: API Routes

## Overview

Hono-based REST API routes for the backend Worker. Handles session management, file uploads, estimate retrieval, reclassification, and cost code queries.

## Goals

- Implement all REST endpoints
- Handle multipart file uploads
- Manage async processing with waitUntil
- Return consistent response formats
- Implement proper error handling

## File Structure

```
packages/backend/src/
├── index.ts              # Worker entry + main router
├── routes/
│   ├── sessions.ts       # Session endpoints
│   ├── estimates.ts      # Estimate endpoints
│   └── cost-codes.ts     # Cost code endpoints
├── middleware/
│   ├── error-handler.ts  # Global error handling
│   └── cors.ts           # CORS configuration
└── lib/
    └── response.ts       # Response helpers
```

## API Endpoints Summary

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/sessions` | Create new session |
| POST | `/api/sessions/:id/upload` | Upload PDF files |
| GET | `/api/sessions/:id/status` | Poll processing status |
| GET | `/api/sessions/:id` | Get full session with comparison |
| GET | `/api/estimates/:id` | Get single estimate detail |
| PATCH | `/api/estimates/:id/items/:itemId` | Reclassify line item |
| GET | `/api/cost-codes/:formatId` | Get cost codes for format |
| GET | `/api/health` | Health check |

## Core Implementation

### Worker Entry (index.ts)

```typescript
import { Hono } from 'hono';
import { cors } from 'hono/cors';
import { sessionsRouter } from './routes/sessions';
import { estimatesRouter } from './routes/estimates';
import { costCodesRouter } from './routes/cost-codes';
import { errorHandler } from './middleware/error-handler';

export interface Env {
  DB: D1Database;
  R2: R2Bucket;
  AI: Ai;
}

// Create app with typed bindings
const app = new Hono<{ Bindings: Env }>();

// Global middleware
app.use('*', cors({
  origin: '*',  // Configure for production
  allowMethods: ['GET', 'POST', 'PATCH', 'DELETE', 'OPTIONS'],
  allowHeaders: ['Content-Type'],
  maxAge: 86400,
}));

// Error handling
app.onError(errorHandler);

// Health check
app.get('/api/health', (c) => {
  return c.json({ 
    status: 'ok', 
    timestamp: new Date().toISOString() 
  });
});

// Mount routers
app.route('/api/sessions', sessionsRouter);
app.route('/api/estimates', estimatesRouter);
app.route('/api/cost-codes', costCodesRouter);

// 404 handler
app.notFound((c) => {
  return c.json({ error: 'Not found' }, 404);
});

export default app;
```

### Sessions Router (routes/sessions.ts)

```typescript
import { Hono } from 'hono';
import type { Env } from '../index';
import {
  createSession,
  getSession,
  updateSessionStatus,
  getSessionWithEstimates,
  createEstimate,
} from '../db/queries';
import { processSession } from '../services/processor';

export const sessionsRouter = new Hono<{ Bindings: Env }>();

/**
 * POST /api/sessions
 * Create a new comparison session
 */
sessionsRouter.post('/', async (c) => {
  try {
    const body = await c.req.json<{ formatId?: string }>().catch(() => ({}));
    const formatId = body.formatId || 'residential-v1';
    
    const sessionId = crypto.randomUUID();
    
    await createSession(c.env.DB, sessionId, formatId);
    
    return c.json({ 
      sessionId, 
      formatId,
      status: 'uploading',
    }, 201);
    
  } catch (error) {
    console.error('Create session error:', error);
    return c.json({ error: 'Failed to create session' }, 500);
  }
});

/**
 * POST /api/sessions/:id/upload
 * Upload PDF files to a session
 */
sessionsRouter.post('/:id/upload', async (c) => {
  const sessionId = c.req.param('id');
  
  try {
    // Verify session exists and is in uploading state
    const session = await getSession(c.env.DB, sessionId);
    
    if (!session) {
      return c.json({ error: 'Session not found' }, 404);
    }
    
    if (session.status !== 'uploading') {
      return c.json({ 
        error: 'Session is not accepting uploads',
        currentStatus: session.status,
      }, 400);
    }
    
    // Parse multipart form data
    const formData = await c.req.formData();
    const files = formData.getAll('files') as File[];
    
    if (files.length === 0) {
      return c.json({ error: 'No files provided' }, 400);
    }
    
    // Validate and filter files
    const validFiles: File[] = [];
    const errors: string[] = [];
    
    for (const file of files) {
      // Check file type
      if (!file.name.toLowerCase().endsWith('.pdf')) {
        errors.push(`${file.name}: Only PDF files are supported`);
        continue;
      }
      
      // Check file size (max 20MB)
      if (file.size > 20 * 1024 * 1024) {
        errors.push(`${file.name}: File exceeds 20MB limit`);
        continue;
      }
      
      // Check for empty file
      if (file.size === 0) {
        errors.push(`${file.name}: File is empty`);
        continue;
      }
      
      validFiles.push(file);
    }
    
    if (validFiles.length === 0) {
      return c.json({ 
        error: 'No valid files to upload',
        details: errors,
      }, 400);
    }
    
    // Store files and create estimate records
    const estimates: Array<{ id: string; filename: string }> = [];
    
    for (const file of validFiles) {
      const estimateId = crypto.randomUUID();
      const r2Key = `${sessionId}/${estimateId}/${file.name}`;
      
      // Store in R2
      const arrayBuffer = await file.arrayBuffer();
      await c.env.R2.put(r2Key, arrayBuffer, {
        customMetadata: {
          filename: file.name,
          uploadedAt: new Date().toISOString(),
        },
      });
      
      // Create estimate record
      await createEstimate(c.env.DB, {
        id: estimateId,
        sessionId,
        filename: file.name,
        r2Key,
      });
      
      estimates.push({ id: estimateId, filename: file.name });
    }
    
    // Update session status
    await updateSessionStatus(c.env.DB, sessionId, 'processing', {
      total: estimates.length,
      completed: 0,
    });
    
    // Start async processing
    c.executionCtx.waitUntil(
      processSession(c.env, sessionId, estimates.map(e => e.id))
    );
    
    return c.json({
      sessionId,
      estimateCount: estimates.length,
      estimates,
      warnings: errors.length > 0 ? errors : undefined,
    });
    
  } catch (error) {
    console.error('Upload error:', error);
    return c.json({ error: 'Upload failed' }, 500);
  }
});

/**
 * GET /api/sessions/:id/status
 * Get session processing status (for polling)
 */
sessionsRouter.get('/:id/status', async (c) => {
  const sessionId = c.req.param('id');
  
  try {
    const session = await getSession(c.env.DB, sessionId);
    
    if (!session) {
      return c.json({ error: 'Session not found' }, 404);
    }
    
    // Get estimate statuses
    const estimates = await c.env.DB.prepare(`
      SELECT id, filename, contractor_name, grand_total, status, error_message
      FROM estimates
      WHERE session_id = ?
      ORDER BY created_at
    `).bind(sessionId).all();
    
    return c.json({
      id: session.id,
      status: session.status,
      progress: session.progress,
      ready: session.status === 'ready',
      errorMessage: session.errorMessage,
      estimates: estimates.results.map((e: any) => ({
        id: e.id,
        filename: e.filename,
        contractorName: e.contractor_name,
        grandTotal: e.grand_total,
        status: e.status,
        errorMessage: e.error_message,
      })),
    });
    
  } catch (error) {
    console.error('Status check error:', error);
    return c.json({ error: 'Failed to get status' }, 500);
  }
});

/**
 * GET /api/sessions/:id
 * Get full session data with estimates and comparison
 */
sessionsRouter.get('/:id', async (c) => {
  const sessionId = c.req.param('id');
  
  try {
    const { session, estimates, comparison } = await getSessionWithEstimates(
      c.env.DB,
      sessionId
    );
    
    if (!session) {
      return c.json({ error: 'Session not found' }, 404);
    }
    
    return c.json({
      session,
      estimates,
      comparison,
    });
    
  } catch (error) {
    console.error('Get session error:', error);
    return c.json({ error: 'Failed to get session' }, 500);
  }
});
```

### Estimates Router (routes/estimates.ts)

```typescript
import { Hono } from 'hono';
import type { Env } from '../index';
import {
  getEstimate,
  getSession,
  updateLineItemClassification,
  getEstimatesBySession,
  saveComparison,
  getCostCodesByFormat,
} from '../db/queries';
import { ComparisonGenerator } from '../services/comparison';

export const estimatesRouter = new Hono<{ Bindings: Env }>();

/**
 * GET /api/estimates/:id
 * Get single estimate with all details
 */
estimatesRouter.get('/:id', async (c) => {
  const estimateId = c.req.param('id');
  
  try {
    const estimate = await getEstimate(c.env.DB, estimateId);
    
    if (!estimate) {
      return c.json({ error: 'Estimate not found' }, 404);
    }
    
    return c.json({ estimate });
    
  } catch (error) {
    console.error('Get estimate error:', error);
    return c.json({ error: 'Failed to get estimate' }, 500);
  }
});

/**
 * PATCH /api/estimates/:id/items/:itemId
 * Reclassify a line item and regenerate comparison
 */
estimatesRouter.patch('/:id/items/:itemId', async (c) => {
  const estimateId = c.req.param('id');
  const lineItemId = c.req.param('itemId');
  
  try {
    // Parse request body
    const body = await c.req.json<{ newCode: string }>();
    
    if (!body.newCode || typeof body.newCode !== 'string') {
      return c.json({ error: 'newCode is required' }, 400);
    }
    
    // Get estimate to find session
    const estimate = await getEstimate(c.env.DB, estimateId);
    
    if (!estimate) {
      return c.json({ error: 'Estimate not found' }, 404);
    }
    
    // Verify line item exists
    const lineItem = estimate.lineItems.find(li => li.id === lineItemId);
    
    if (!lineItem) {
      return c.json({ error: 'Line item not found' }, 404);
    }
    
    // Validate new code exists
    const session = await getSession(c.env.DB, estimate.sessionId);
    const costCodes = await getCostCodesByFormat(c.env.DB, session!.formatId);
    
    const validCode = costCodes.find(cc => cc.code === body.newCode);
    if (!validCode) {
      return c.json({ error: 'Invalid cost code' }, 400);
    }
    
    // Update line item classification
    await updateLineItemClassification(
      c.env.DB,
      lineItemId,
      body.newCode,
      true  // isManualOverride
    );
    
    // Regenerate comparison
    const allEstimates = await getEstimatesBySession(c.env.DB, estimate.sessionId);
    
    // Update the line item in memory for comparison generation
    const updatedEstimates = allEstimates.map(est => {
      if (est.id === estimateId) {
        return {
          ...est,
          lineItems: est.lineItems.map(li =>
            li.id === lineItemId
              ? { ...li, costCode: body.newCode, confidence: 1.0, isManualOverride: true }
              : li
          ),
        };
      }
      return est;
    });
    
    const generator = new ComparisonGenerator(costCodes);
    const comparison = generator.generate(updatedEstimates);
    
    await saveComparison(c.env.DB, comparison);
    
    // Get updated line item
    const updatedItem = {
      ...lineItem,
      costCode: body.newCode,
      confidence: 1.0,
      isManualOverride: true,
    };
    
    return c.json({
      lineItem: updatedItem,
      comparison,
      sessionId: estimate.sessionId,
    });
    
  } catch (error) {
    console.error('Reclassify error:', error);
    return c.json({ error: 'Failed to reclassify' }, 500);
  }
});
```

### Cost Codes Router (routes/cost-codes.ts)

```typescript
import { Hono } from 'hono';
import type { Env } from '../index';
import { getCostCodesByFormat } from '../db/queries';
import { buildFullLabel } from '@estimate-compare/shared';

export const costCodesRouter = new Hono<{ Bindings: Env }>();

/**
 * GET /api/cost-codes/:formatId
 * Get all cost codes for a format
 */
costCodesRouter.get('/:formatId', async (c) => {
  const formatId = c.req.param('formatId');
  
  try {
    const codes = await getCostCodesByFormat(c.env.DB, formatId);
    
    if (codes.length === 0) {
      return c.json({ error: 'Format not found' }, 404);
    }
    
    // Add full labels
    const codesWithLabels = codes.map(code => ({
      ...code,
      fullLabel: buildFullLabel(code, codes),
    }));
    
    return c.json({ codes: codesWithLabels });
    
  } catch (error) {
    console.error('Get cost codes error:', error);
    return c.json({ error: 'Failed to get cost codes' }, 500);
  }
});

/**
 * GET /api/cost-codes
 * List available formats
 */
costCodesRouter.get('/', async (c) => {
  try {
    const formats = await c.env.DB.prepare(`
      SELECT id, name, version, description
      FROM cost_code_formats
      ORDER BY name
    `).all();
    
    return c.json({ formats: formats.results });
    
  } catch (error) {
    console.error('List formats error:', error);
    return c.json({ error: 'Failed to list formats' }, 500);
  }
});
```

### Error Handler Middleware (middleware/error-handler.ts)

```typescript
import type { Context } from 'hono';

export function errorHandler(err: Error, c: Context) {
  console.error('Unhandled error:', err);
  
  // Check for specific error types
  if (err.name === 'ValidationError') {
    return c.json({
      error: 'Validation failed',
      message: err.message,
    }, 400);
  }
  
  if (err.name === 'NotFoundError') {
    return c.json({
      error: 'Not found',
      message: err.message,
    }, 404);
  }
  
  // Default 500 error
  return c.json({
    error: 'Internal server error',
    message: process.env.NODE_ENV === 'development' ? err.message : undefined,
  }, 500);
}
```

### Response Helpers (lib/response.ts)

```typescript
import type { Context } from 'hono';

export interface ApiResponse<T> {
  data?: T;
  error?: string;
  message?: string;
}

export function success<T>(c: Context, data: T, status: number = 200) {
  return c.json({ data }, status);
}

export function error(c: Context, message: string, status: number = 400) {
  return c.json({ error: message }, status);
}

export function notFound(c: Context, resource: string = 'Resource') {
  return c.json({ error: `${resource} not found` }, 404);
}

export function validationError(c: Context, details: string | string[]) {
  return c.json({
    error: 'Validation failed',
    details: Array.isArray(details) ? details : [details],
  }, 400);
}
```

### Background Processor (services/processor.ts)

```typescript
import type { Env } from '../index';
import { extractPDFFromR2 } from './pdf-extractor';
import { EstimateParser } from './estimate-parser';
import { Classifier } from './classifier';
import { ComparisonGenerator } from './comparison';
import {
  getSession,
  updateSessionStatus,
  updateEstimate,
  createLineItem,
  createNote,
  getCostCodesByFormat,
  getEstimatesBySession,
  saveComparison,
} from '../db/queries';

/**
 * Process all estimates in a session
 */
export async function processSession(
  env: Env,
  sessionId: string,
  estimateIds: string[]
): Promise<void> {
  const parser = new EstimateParser(env.AI);
  const classifier = new Classifier(env.AI);
  
  // Load cost codes once
  const session = await getSession(env.DB, sessionId);
  if (!session) {
    console.error(`Session ${sessionId} not found`);
    return;
  }
  
  const costCodes = await getCostCodesByFormat(env.DB, session.formatId);
  
  // Process each estimate
  for (let i = 0; i < estimateIds.length; i++) {
    const estimateId = estimateIds[i];
    
    try {
      await processEstimate(env, estimateId, parser, classifier, costCodes);
      
      // Update progress
      await updateSessionStatus(env.DB, sessionId, 'processing', {
        completed: i + 1,
      });
      
    } catch (error) {
      console.error(`Error processing estimate ${estimateId}:`, error);
      
      await updateEstimate(env.DB, estimateId, {
        status: 'error',
        errorMessage: error instanceof Error ? error.message : 'Unknown error',
      });
    }
  }
  
  // Generate comparison
  try {
    const estimates = await getEstimatesBySession(env.DB, sessionId);
    const generator = new ComparisonGenerator(costCodes);
    const comparison = generator.generate(estimates);
    
    await saveComparison(env.DB, comparison);
    
    // Mark session as ready
    await updateSessionStatus(env.DB, sessionId, 'ready', {
      completed: estimateIds.length,
      currentFile: null,
    });
    
  } catch (error) {
    console.error(`Error generating comparison for ${sessionId}:`, error);
    
    await updateSessionStatus(env.DB, sessionId, 'error', undefined,
      error instanceof Error ? error.message : 'Comparison generation failed'
    );
  }
}

/**
 * Process a single estimate
 */
async function processEstimate(
  env: Env,
  estimateId: string,
  parser: EstimateParser,
  classifier: Classifier,
  costCodes: CostCode[]
): Promise<void> {
  // Get estimate record
  const estimate = await env.DB.prepare(
    'SELECT * FROM estimates WHERE id = ?'
  ).bind(estimateId).first();
  
  if (!estimate) {
    throw new Error('Estimate not found');
  }
  
  // Update status - extracting
  await updateEstimate(env.DB, estimateId, { status: 'extracting' });
  
  // Extract PDF text
  const extraction = await extractPDFFromR2(env.R2, estimate.r2_key as string);
  
  // Store raw text
  await updateEstimate(env.DB, estimateId, {
    rawText: extraction.fullText.slice(0, 50000),
    status: 'classifying',
  });
  
  // Parse with LLM
  const parsed = await parser.parse(extraction.fullText, estimate.filename as string);
  
  // Create line items with IDs
  const lineItemsWithIds = parsed.lineItems.map((item, idx) => ({
    ...item,
    id: crypto.randomUUID(),
    lineNumber: idx + 1,
  }));
  
  // Classify line items
  const classifications = await classifier.classifyLineItems(
    lineItemsWithIds.map(item => ({
      id: item.id,
      description: item.description,
      sectionHeader: item.sectionHeader,
    })),
    costCodes
  );
  
  // Update estimate with parsed data
  await updateEstimate(env.DB, estimateId, {
    contractorName: parsed.contractor.name,
    contractorAddress: parsed.contractor.address,
    contractorPhone: parsed.contractor.phone,
    contractorEmail: parsed.contractor.email,
    clientName: parsed.client?.name || null,
    clientProjectName: parsed.client?.projectName || null,
    subtotal: parsed.documentTotals.subtotal,
    tax: parsed.documentTotals.tax,
    grandTotal: parsed.documentTotals.grandTotal,
    status: 'ready',
  });
  
  // Insert line items
  for (const item of lineItemsWithIds) {
    const classification = classifications.get(item.id);
    
    await createLineItem(env.DB, {
      id: item.id,
      estimateId,
      description: item.description,
      sectionHeader: item.sectionHeader,
      quantity: item.quantity,
      unit: item.unit,
      unitPrice: item.unitPrice,
      totalPrice: item.totalPrice,
      lineNumber: item.lineNumber,
      costCode: classification?.code || null,
      confidence: classification?.confidence || 0,
      alternateCodes: classification?.alternates || [],
    });
  }
  
  // Insert notes
  for (const note of parsed.notes) {
    await createNote(env.DB, estimateId, note.noteType, note.content);
  }
}
```

## Todo List

### Worker Entry

- [ ] Create index.ts with Hono app
- [ ] Configure CORS middleware
- [ ] Setup error handler
- [ ] Mount all routers
- [ ] Add health check endpoint
- [ ] Add 404 handler

### Sessions Router

- [ ] Create routes/sessions.ts
- [ ] Implement POST /sessions (create)
- [ ] Implement POST /sessions/:id/upload
- [ ] Validate file types and sizes
- [ ] Store files in R2
- [ ] Create estimate records
- [ ] Trigger async processing with waitUntil
- [ ] Implement GET /sessions/:id/status
- [ ] Implement GET /sessions/:id

### Estimates Router

- [ ] Create routes/estimates.ts
- [ ] Implement GET /estimates/:id
- [ ] Implement PATCH /estimates/:id/items/:itemId
- [ ] Validate new cost code
- [ ] Update line item in database
- [ ] Regenerate comparison
- [ ] Return updated data

### Cost Codes Router

- [ ] Create routes/cost-codes.ts
- [ ] Implement GET /cost-codes/:formatId
- [ ] Implement GET /cost-codes (list formats)
- [ ] Add full labels to codes

### Middleware

- [ ] Create middleware/error-handler.ts
- [ ] Handle different error types
- [ ] Log errors appropriately
- [ ] Return safe error messages

### Background Processing

- [ ] Create services/processor.ts
- [ ] Implement processSession function
- [ ] Implement processEstimate function
- [ ] Handle errors per-estimate
- [ ] Update progress during processing
- [ ] Generate comparison after all done

### Testing

- [ ] Test create session
- [ ] Test file upload (single file)
- [ ] Test file upload (multiple files)
- [ ] Test file validation (type, size)
- [ ] Test status polling
- [ ] Test get session data
- [ ] Test get estimate
- [ ] Test reclassification
- [ ] Test comparison regeneration
- [ ] Test error handling
- [ ] Test concurrent uploads

## Verification Checklist

- [ ] All endpoints return JSON
- [ ] Error responses are consistent
- [ ] File uploads work with multipart
- [ ] Async processing completes successfully
- [ ] Status polling returns accurate info
- [ ] Reclassification updates both DB and comparison
- [ ] CORS allows frontend access
- [ ] Large files are rejected appropriately

## Notes

### Response Formats

**Success Response:**
```json
{
  "sessionId": "abc123",
  "status": "ready",
  "data": { ... }
}
```

**Error Response:**
```json
{
  "error": "Error message",
  "details": ["Additional info"]
}
```

### File Upload Limits

| Constraint | Limit |
|------------|-------|
| Max file size | 20MB |
| File types | PDF only |
| Max files per upload | 10 |

### Processing Flow

```
POST /sessions → Session created (status: uploading)
     ↓
POST /sessions/:id/upload → Files stored, processing starts
     ↓
GET /sessions/:id/status → Poll until status: ready
     ↓
GET /sessions/:id → Get full data with comparison
```

### Reclassification Flow

```
PATCH /estimates/:id/items/:itemId
     ↓
Update line_items table (cost_code, is_manual_override)
     ↓
Regenerate comparison
     ↓
Return updated line item + new comparison
```

## Time Estimate

| Task | Estimate |
|------|----------|
| Worker entry | 30 min |
| Sessions router | 2 hours |
| Estimates router | 1.5 hours |
| Cost codes router | 30 min |
| Middleware | 30 min |
| Background processor | 1.5 hours |
| Testing | 1.5 hours |
| **Total** | **~8 hours** |
