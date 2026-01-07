# API Reference

Complete API documentation for the Estimate Compare backend.

## Base URL

```
Development: http://localhost:8787/api
Production:  https://estimate-compare-api.<your-domain>.workers.dev/api
```

## Authentication

Currently, the API does not require authentication (MVP). Future versions will implement session-based or token authentication.

## Response Format

All responses are JSON with the following structure:

**Success Response:**
```json
{
  "data": { ... }
}
```

**Error Response:**
```json
{
  "error": "Error message",
  "code": "ERROR_CODE",
  "details": { ... }
}
```

## HTTP Status Codes

| Code | Description |
|------|-------------|
| 200 | Success |
| 201 | Created |
| 400 | Bad Request - Invalid input |
| 404 | Not Found |
| 500 | Internal Server Error |

---

## Endpoints

### Health Check

#### GET /api/health

Check if the API is running.

**Response:**
```json
{
  "status": "ok",
  "timestamp": "2024-01-15T10:00:00Z"
}
```

---

## Sessions

### Create Session

#### POST /api/sessions

Create a new comparison session.

**Request Body:**
```json
{
  "formatId": "residential-v1"  // Optional, defaults to "residential-v1"
}
```

**Response:** `201 Created`
```json
{
  "sessionId": "550e8400-e29b-41d4-a716-446655440000",
  "formatId": "residential-v1"
}
```

**Example:**
```bash
curl -X POST http://localhost:8787/api/sessions \
  -H "Content-Type: application/json" \
  -d '{"formatId": "residential-v1"}'
```

---

### Upload Estimates

#### POST /api/sessions/:sessionId/upload

Upload PDF files to a session for processing.

**URL Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| sessionId | string | Session UUID |

**Request:**
- Content-Type: `multipart/form-data`
- Field name: `files` (multiple files allowed)
- Accepted: PDF files only

**Response:** `200 OK`
```json
{
  "sessionId": "550e8400-e29b-41d4-a716-446655440000",
  "estimateCount": 3,
  "estimates": [
    {
      "id": "est-001",
      "filename": "contractor_a.pdf"
    },
    {
      "id": "est-002", 
      "filename": "contractor_b.pdf"
    },
    {
      "id": "est-003",
      "filename": "contractor_c.pdf"
    }
  ]
}
```

**Errors:**
| Code | Description |
|------|-------------|
| 404 | Session not found |
| 400 | No files provided |
| 400 | Invalid file type (non-PDF) |

**Example:**
```bash
curl -X POST http://localhost:8787/api/sessions/550e8400-e29b-41d4-a716-446655440000/upload \
  -F "files=@contractor_a.pdf" \
  -F "files=@contractor_b.pdf"
```

---

### Get Session Status

#### GET /api/sessions/:sessionId/status

Poll for session processing status. Use this endpoint while processing is in progress.

**URL Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| sessionId | string | Session UUID |

**Response:** `200 OK`
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "processing",
  "progress": {
    "total": 3,
    "completed": 1,
    "currentFile": "contractor_b.pdf"
  },
  "ready": false,
  "estimates": [
    {
      "id": "est-001",
      "filename": "contractor_a.pdf",
      "contractorName": "ABC Construction",
      "grandTotal": 45000.00,
      "status": "ready",
      "errorMessage": null
    },
    {
      "id": "est-002",
      "filename": "contractor_b.pdf",
      "contractorName": null,
      "grandTotal": null,
      "status": "classifying",
      "errorMessage": null
    },
    {
      "id": "est-003",
      "filename": "contractor_c.pdf",
      "contractorName": null,
      "grandTotal": null,
      "status": "pending",
      "errorMessage": null
    }
  ]
}
```

**Status Values:**

Session Status:
| Status | Description |
|--------|-------------|
| `uploading` | Session created, waiting for files |
| `processing` | Files uploaded, processing in progress |
| `ready` | All estimates processed, comparison available |
| `error` | Processing failed |

Estimate Status:
| Status | Description |
|--------|-------------|
| `pending` | Waiting to be processed |
| `extracting` | Extracting text from PDF |
| `classifying` | Parsing and classifying line items |
| `ready` | Processing complete |
| `error` | Processing failed for this estimate |

**Polling Recommendation:**
Poll every 1 second until `ready` is `true`.

**Example:**
```bash
curl http://localhost:8787/api/sessions/550e8400-e29b-41d4-a716-446655440000/status
```

---

### Get Full Session

#### GET /api/sessions/:sessionId

Get complete session data including all estimates and comparison.

**URL Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| sessionId | string | Session UUID |

**Response:** `200 OK`
```json
{
  "session": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "formatId": "residential-v1",
    "status": "ready",
    "progress": {
      "total": 2,
      "completed": 2,
      "currentFile": null
    },
    "errorMessage": null,
    "createdAt": "2024-01-15T10:00:00Z",
    "updatedAt": "2024-01-15T10:05:00Z"
  },
  "estimates": [
    {
      "id": "est-001",
      "sessionId": "550e8400-e29b-41d4-a716-446655440000",
      "filename": "contractor_a.pdf",
      "contractorName": "ABC Construction",
      "grandTotal": 45000.00,
      "status": "ready",
      "contractor": {
        "name": "ABC Construction",
        "address": "123 Main St, Anytown, USA",
        "phone": "(555) 123-4567",
        "email": "info@abcconstruction.com"
      },
      "client": {
        "name": "John Smith",
        "projectName": "Kitchen Renovation"
      },
      "documentTotals": {
        "subtotal": 42000.00,
        "tax": 3000.00,
        "grandTotal": 45000.00
      },
      "lineItems": [
        {
          "id": "item-001",
          "estimateId": "est-001",
          "description": "Remove existing cabinets",
          "sectionHeader": "Demolition",
          "quantity": 1,
          "unit": "LS",
          "unitPrice": 500.00,
          "totalPrice": 500.00,
          "lineNumber": 1,
          "costCode": "00_L",
          "confidence": 0.85,
          "isManualOverride": false,
          "alternateCodes": [
            { "code": "00_S", "confidence": 0.6 }
          ]
        }
        // ... more line items
      ],
      "notes": [
        {
          "id": 1,
          "noteType": "exclusion",
          "content": "Permits not included"
        }
      ]
    }
    // ... more estimates
  ],
  "comparison": {
    "id": "comp-001",
    "sessionId": "550e8400-e29b-41d4-a716-446655440000",
    "generatedAt": "2024-01-15T10:05:00Z",
    "rows": [
      {
        "code": "00_L",
        "label": "Labor",
        "fullLabel": "Demolition → Labor",
        "level": 2,
        "isSubtotal": false,
        "amounts": {
          "est-001": {
            "total": 500.00,
            "lineItemIds": ["item-001"],
            "lineItemCount": 1
          },
          "est-002": {
            "total": 750.00,
            "lineItemIds": ["item-101"],
            "lineItemCount": 1
          }
        },
        "computed": {
          "min": 500.00,
          "max": 750.00,
          "average": 625.00,
          "range": 250.00,
          "rangePercent": 0.4
        }
      }
      // ... more rows
    ],
    "summary": {
      "byEstimate": {
        "est-001": 45000.00,
        "est-002": 48000.00
      },
      "range": {
        "min": 45000.00,
        "max": 48000.00,
        "spread": 3000.00
      },
      "average": 46500.00
    }
  }
}
```

**Errors:**
| Code | Description |
|------|-------------|
| 404 | Session not found |

**Example:**
```bash
curl http://localhost:8787/api/sessions/550e8400-e29b-41d4-a716-446655440000
```

---

## Estimates

### Get Single Estimate

#### GET /api/estimates/:estimateId

Get detailed data for a single estimate.

**URL Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| estimateId | string | Estimate UUID |

**Response:** `200 OK`
```json
{
  "estimate": {
    "id": "est-001",
    "sessionId": "550e8400-e29b-41d4-a716-446655440000",
    "filename": "contractor_a.pdf",
    "contractorName": "ABC Construction",
    "grandTotal": 45000.00,
    "status": "ready",
    "contractor": {
      "name": "ABC Construction",
      "address": "123 Main St, Anytown, USA",
      "phone": "(555) 123-4567",
      "email": "info@abcconstruction.com"
    },
    "client": {
      "name": "John Smith",
      "projectName": "Kitchen Renovation"
    },
    "documentTotals": {
      "subtotal": 42000.00,
      "tax": 3000.00,
      "grandTotal": 45000.00
    },
    "lineItems": [
      {
        "id": "item-001",
        "estimateId": "est-001",
        "description": "Remove existing cabinets",
        "sectionHeader": "Demolition",
        "quantity": 1,
        "unit": "LS",
        "unitPrice": 500.00,
        "totalPrice": 500.00,
        "lineNumber": 1,
        "costCode": "00_L",
        "confidence": 0.85,
        "isManualOverride": false,
        "alternateCodes": [
          { "code": "00_S", "confidence": 0.6 }
        ]
      }
      // ... more line items
    ],
    "notes": [
      {
        "id": 1,
        "noteType": "exclusion",
        "content": "Permits not included"
      }
    ],
    "rawText": "ABC CONSTRUCTION\n123 Main St..."
  }
}
```

**Errors:**
| Code | Description |
|------|-------------|
| 404 | Estimate not found |

**Example:**
```bash
curl http://localhost:8787/api/estimates/est-001
```

---

### Reclassify Line Item

#### PATCH /api/estimates/:estimateId/items/:lineItemId

Manually reclassify a line item to a different cost code.

**URL Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| estimateId | string | Estimate UUID |
| lineItemId | string | Line item UUID |

**Request Body:**
```json
{
  "newCode": "03_M"
}
```

**Response:** `200 OK`
```json
{
  "lineItem": {
    "id": "item-001",
    "estimateId": "est-001",
    "description": "2x4 Lumber",
    "sectionHeader": "Framing",
    "quantity": 100,
    "unit": "BF",
    "unitPrice": 5.00,
    "totalPrice": 500.00,
    "lineNumber": 5,
    "costCode": "03_M",
    "confidence": 1.0,
    "isManualOverride": true,
    "alternateCodes": []
  },
  "comparison": {
    // Full regenerated comparison object
    // (same structure as in GET /api/sessions/:id)
  },
  "sessionId": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Errors:**
| Code | Description |
|------|-------------|
| 404 | Estimate not found |
| 404 | Line item not found |
| 400 | Invalid cost code |

**Example:**
```bash
curl -X PATCH http://localhost:8787/api/estimates/est-001/items/item-001 \
  -H "Content-Type: application/json" \
  -d '{"newCode": "03_M"}'
```

---

## Cost Codes

### Get Cost Codes

#### GET /api/cost-codes/:formatId

Get all cost codes for a classification format.

**URL Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| formatId | string | Format ID (e.g., "residential-v1") |

**Response:** `200 OK`
```json
{
  "format": {
    "id": "residential-v1",
    "name": "Residential Construction",
    "version": "1.0",
    "description": "Standard cost codes for residential construction projects"
  },
  "codes": [
    {
      "code": "00",
      "label": "Demolition",
      "level": 1,
      "parentCode": null,
      "sortOrder": 0,
      "keywords": ["demo", "tear out", "removal"],
      "description": null
    },
    {
      "code": "00_L",
      "label": "Labor",
      "level": 2,
      "parentCode": "00",
      "sortOrder": 0,
      "keywords": ["labor", "hours", "crew"],
      "description": null
    },
    {
      "code": "00_M",
      "label": "Material",
      "level": 2,
      "parentCode": "00",
      "sortOrder": 1,
      "keywords": ["material", "supplies"],
      "description": null
    },
    {
      "code": "00_S",
      "label": "Subcontracts",
      "level": 2,
      "parentCode": "00",
      "sortOrder": 2,
      "keywords": ["subcontract", "sub"],
      "description": null
    }
    // ... more codes (64 total for residential-v1)
  ]
}
```

**Errors:**
| Code | Description |
|------|-------------|
| 404 | Format not found |

**Example:**
```bash
curl http://localhost:8787/api/cost-codes/residential-v1
```

---

### List Available Formats

#### GET /api/cost-codes

List all available cost code formats.

**Response:** `200 OK`
```json
{
  "formats": [
    {
      "id": "residential-v1",
      "name": "Residential Construction",
      "version": "1.0",
      "description": "Standard cost codes for residential construction projects"
    }
    // ... more formats if available
  ]
}
```

**Example:**
```bash
curl http://localhost:8787/api/cost-codes
```

---

## Data Types Reference

### Session

```typescript
interface Session {
  id: string;                           // UUID
  formatId: string;                     // Cost code format ID
  status: 'uploading' | 'processing' | 'ready' | 'error';
  progress: {
    total: number;                      // Total estimates
    completed: number;                  // Processed estimates
    currentFile: string | null;         // Currently processing
  };
  errorMessage: string | null;
  createdAt: string;                    // ISO 8601
  updatedAt: string;                    // ISO 8601
}
```

### Estimate

```typescript
interface Estimate {
  id: string;                           // UUID
  sessionId: string;                    // Parent session UUID
  filename: string;                     // Original filename
  contractorName: string | null;        // Extracted contractor name
  grandTotal: number | null;            // Total amount
  status: 'pending' | 'extracting' | 'classifying' | 'ready' | 'error';
  errorMessage: string | null;
  
  contractor: {
    name: string | null;
    address: string | null;
    phone: string | null;
    email: string | null;
  };
  
  client: {
    name: string | null;
    projectName: string | null;
  } | null;
  
  documentTotals: {
    subtotal: number | null;
    tax: number | null;
    grandTotal: number | null;
  };
  
  lineItems: LineItem[];
  notes: EstimateNote[];
  rawText?: string;                     // Extracted PDF text (debug)
}
```

### LineItem

```typescript
interface LineItem {
  id: string;                           // UUID
  estimateId: string;                   // Parent estimate UUID
  
  // Extracted data
  description: string;                  // Line item description
  sectionHeader: string | null;         // Section it belongs to
  quantity: number | null;
  unit: string | null;                  // SF, LF, EA, HR, LS, etc.
  unitPrice: number | null;
  totalPrice: number;                   // Required
  lineNumber: number | null;
  
  // Classification
  costCode: string | null;              // Assigned cost code
  confidence: number;                   // 0-1 confidence score
  isManualOverride: boolean;            // Was manually reclassified
  alternateCodes: Array<{
    code: string;
    confidence: number;
  }>;
}
```

### EstimateNote

```typescript
interface EstimateNote {
  id: number;
  noteType: 'note' | 'exclusion' | 'disclaimer' | 'term';
  content: string;
}
```

### Comparison

```typescript
interface Comparison {
  id: string;                           // UUID
  sessionId: string;                    // Parent session UUID
  generatedAt: string;                  // ISO 8601
  rows: ComparisonRow[];
  summary: ComparisonSummary;
}

interface ComparisonRow {
  code: string;                         // Cost code or special (GRAND_TOTAL)
  label: string;                        // Short label
  fullLabel: string;                    // Full path label
  level: number;                        // 0=total, 1=division, 2=cost type
  isSubtotal: boolean;                  // Is this a subtotal row
  
  amounts: Record<string, ComparisonCell | null>;  // By estimate ID
  computed: ComputedStats;
}

interface ComparisonCell {
  total: number;                        // Sum of line items
  lineItemIds: string[];                // Contributing line items
  lineItemCount: number;
}

interface ComputedStats {
  min: number | null;
  max: number | null;
  average: number | null;
  range: number | null;                 // max - min
  rangePercent: number | null;          // range / average
}

interface ComparisonSummary {
  byEstimate: Record<string, number>;   // Total by estimate ID
  range: {
    min: number;
    max: number;
    spread: number;
  };
  average: number;
}
```

### CostCode

```typescript
interface CostCode {
  code: string;                         // Unique code (e.g., "03_L")
  label: string;                        // Display label
  level: number;                        // 1 = division, 2 = cost type
  parentCode: string | null;            // Parent code reference
  sortOrder: number;                    // Display order
  keywords: string[] | null;            // Classification hints
  description: string | null;
}
```

---

## Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `SESSION_NOT_FOUND` | 404 | Session with given ID does not exist |
| `ESTIMATE_NOT_FOUND` | 404 | Estimate with given ID does not exist |
| `LINE_ITEM_NOT_FOUND` | 404 | Line item with given ID does not exist |
| `FORMAT_NOT_FOUND` | 404 | Cost code format does not exist |
| `INVALID_COST_CODE` | 400 | Provided cost code is not valid |
| `NO_FILES_PROVIDED` | 400 | Upload request has no files |
| `INVALID_FILE_TYPE` | 400 | File is not a PDF |
| `PROCESSING_FAILED` | 500 | Background processing failed |
| `PDF_EXTRACTION_FAILED` | 500 | Could not extract text from PDF |
| `PARSE_FAILED` | 500 | Could not parse estimate content |

---

## Typical Workflow

### 1. Create Session and Upload

```javascript
// Create session
const { sessionId } = await fetch('/api/sessions', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ formatId: 'residential-v1' })
}).then(r => r.json());

// Upload files
const formData = new FormData();
files.forEach(file => formData.append('files', file));

await fetch(`/api/sessions/${sessionId}/upload`, {
  method: 'POST',
  body: formData
});
```

### 2. Poll for Completion

```javascript
async function waitForProcessing(sessionId) {
  while (true) {
    const status = await fetch(`/api/sessions/${sessionId}/status`)
      .then(r => r.json());
    
    if (status.ready) {
      return status;
    }
    
    if (status.status === 'error') {
      throw new Error(status.errorMessage);
    }
    
    // Wait 1 second before polling again
    await new Promise(r => setTimeout(r, 1000));
  }
}
```

### 3. Fetch Results

```javascript
const { session, estimates, comparison } = await fetch(
  `/api/sessions/${sessionId}`
).then(r => r.json());
```

### 4. Reclassify Item

```javascript
const { lineItem, comparison } = await fetch(
  `/api/estimates/${estimateId}/items/${lineItemId}`,
  {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ newCode: '03_M' })
  }
).then(r => r.json());
```

---

## Rate Limits

Currently no rate limits are enforced (MVP). Future versions may implement:

- 10 sessions per hour per IP
- 50 MB total upload per session
- 10 estimates per session

---

## CORS

The API allows cross-origin requests from:
- `http://localhost:*` (development)
- Production frontend domains (to be configured)

Headers:
```
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, POST, PATCH, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type
```
