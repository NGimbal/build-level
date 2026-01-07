# Module 14: BAML Integration

## Overview

Integration of [BAML (Boundary AI Markup Language)](https://github.com/BoundaryML/baml) to replace manual LLM prompt engineering and JSON parsing with a type-safe, schema-driven approach. BAML provides compiler-generated clients, automatic JSON repair, and superior developer experience for iterating on extraction accuracy.

## Goals

- Replace fragile JSON parsing with BAML's Schema-Aligned Parsing (SAP)
- Enable type-safe extraction with compiler-generated TypeScript clients
- Improve developer experience with VS Code playground for prompt testing
- Establish foundation for A/B testing and prompt versioning
- Reduce boilerplate for retry logic, validation, and streaming

## Why BAML for Build Level

### Current Pain Points Addressed

| Current Approach | Problem | BAML Solution |
|-----------------|---------|---------------|
| Manual JSON extraction with regex | LLMs add markdown blocks, extra text | SAP algorithm handles malformed output |
| Runtime type validation | Errors discovered at runtime | Compile-time type checking |
| Prompts as TypeScript strings | Hard to diff, no syntax highlighting | `.baml` files with dedicated tooling |
| Manual retry/fallback logic | Boilerplate in every service | Built-in with configurable strategies |
| No prompt versioning | Can't track what changed | Version controlled `.baml` files |
| No testing framework | Test in production | VS Code playground for local testing |

### Key BAML Features

1. **Schema-Aligned Parsing (SAP)** - Extracts structured data even when LLM output contains:
   - Markdown code blocks
   - Chain-of-thought reasoning before JSON
   - Minor syntax errors in JSON
   - Extra explanatory text

2. **Multi-Model Support** - Single schema works across:
   - Cloudflare Workers AI (Llama 3.1)
   - OpenAI (GPT-4, GPT-3.5)
   - Anthropic (Claude)
   - Any OpenAI-compatible endpoint

3. **Type Generation** - Compiles `.baml` files to:
   - TypeScript interfaces
   - Typed client functions
   - Runtime validators

## File Structure

```
build-level/
├── baml_src/                    # BAML source files
│   ├── clients.baml             # LLM client configurations
│   ├── types.baml               # Shared type definitions
│   ├── extraction.baml          # Estimate extraction function
│   ├── classification.baml      # Cost code classification function
│   └── tests/                   # Test cases for playground
│       ├── extraction_tests.baml
│       └── classification_tests.baml
├── baml_client/                 # Generated TypeScript (gitignored)
│   └── index.ts
├── packages/
│   ├── backend/
│   │   └── src/
│   │       └── services/
│   │           ├── estimate-parser.ts    # Updated to use BAML
│   │           └── classifier.ts         # Updated to use BAML
│   └── shared/
│       └── src/
│           └── types/
│               └── extraction.ts         # Import from baml_client
└── baml.config.yaml             # BAML configuration
```

## Core Implementation

### BAML Configuration (baml.config.yaml)

```yaml
# baml.config.yaml
version: 1

# Output directory for generated code
output_dir: ./baml_client

# Language targets
generators:
  - language: typescript
    output_dir: ./baml_client

# Default client for development
default_client: cloudflare-llama
```

### Client Configuration (baml_src/clients.baml)

```baml
// baml_src/clients.baml

// Primary client: Cloudflare Workers AI
client<llm> CloudflareLlama {
  provider openai-generic
  options {
    base_url "https://api.cloudflare.com/client/v4/accounts/${env.CF_ACCOUNT_ID}/ai/v1"
    api_key env.CF_API_TOKEN
    model "@cf/meta/llama-3.1-8b-instruct"
    default_role "user"
  }
}

// Alternative: OpenAI for higher accuracy when needed
client<llm> OpenAIGPT4 {
  provider openai
  options {
    model "gpt-4-turbo-preview"
    api_key env.OPENAI_API_KEY
  }
}

// Alternative: Anthropic Claude
client<llm> AnthropicClaude {
  provider anthropic
  options {
    model "claude-3-sonnet-20240229"
    api_key env.ANTHROPIC_API_KEY
  }
}

// Fallback strategy: try Cloudflare first, then OpenAI
client<llm> ExtractionClient {
  strategy fallback
  options {
    clients [CloudflareLlama, OpenAIGPT4]
  }
}
```

### Type Definitions (baml_src/types.baml)

```baml
// baml_src/types.baml

// Contractor information from estimate header
class Contractor {
  name string? @description("Company or contractor name")
  address string? @description("Full mailing address")
  phone string? @description("Phone number")
  email string? @description("Email address")
}

// Client/customer information
class Client {
  name string? @description("Customer or client name")
  projectName string? @description("Project name or job site address")
}

// Individual line item from estimate
class LineItem {
  description string @description("EXACT text from document - do not paraphrase")
  sectionHeader string? @description("Section this item belongs to (e.g., 'Framing', 'Electrical')")
  quantity float? @description("Numeric quantity if specified")
  unit string? @description("Unit of measure (SF, LF, EA, HR, LS, etc.)")
  unitPrice float? @description("Price per unit if specified")
  totalPrice float @description("Total/extended price - REQUIRED")
}

// Note or exclusion from estimate
enum NoteType {
  Note @alias("note")
  Exclusion @alias("exclusion")
  Disclaimer @alias("disclaimer")
  Term @alias("term")
}

class EstimateNote {
  noteType NoteType
  content string @description("The note text content")
}

// Document totals
class DocumentTotals {
  subtotal float? @description("Subtotal before tax")
  tax float? @description("Tax amount if shown")
  grandTotal float @description("Final total amount due")
}

// Complete parsed estimate
class ParsedEstimate {
  contractor Contractor
  client Client?
  lineItems LineItem[]
  notes EstimateNote[]
  documentTotals DocumentTotals
}

// Classification result for a line item
class ClassificationResult {
  id string @description("Line item ID from input")
  code string @description("Best matching cost code (e.g., '03_L', '12_M')")
  confidence float @description("Confidence score 0.0-1.0")
  alternates AlternateCode[] @description("1-2 alternative codes if uncertain")
}

class AlternateCode {
  code string
  confidence float
}
```

### Extraction Function (baml_src/extraction.baml)

```baml
// baml_src/extraction.baml

function ExtractEstimate(pdfText: string, filename: string) -> ParsedEstimate {
  client ExtractionClient

  prompt #"
    You are a construction estimate parsing expert. Extract structured data from this construction estimate document.

    FILENAME: {{ filename }}

    DOCUMENT TEXT:
    {{ pdfText }}

    CRITICAL EXTRACTION RULES:

    1. LINE ITEMS - Extract EVERY item that has a price:
       - Preserve EXACT original descriptions - do not paraphrase or summarize
       - Include all line items even if they seem like duplicates
       - If an item spans multiple lines, combine into one description
       - totalPrice is REQUIRED - skip items without a discernible price

    2. NUMBERS - Parse all monetary values as plain numbers:
       - Remove $ symbols: "$1,234.56" → 1234.56
       - Remove commas: "1,234" → 1234
       - Handle parentheses as negative: "(500)" → -500

    3. SECTIONS - Identify section headers:
       - Common sections: Demolition, Framing, Electrical, Plumbing, HVAC, etc.
       - Assign items to the most recent section header above them
       - If no clear section, leave sectionHeader as null

    4. NOTES & EXCLUSIONS - Capture separately:
       - "Exclusions:" sections → Exclusion
       - "Notes:" or "Note:" → Note
       - "Terms:" → Term
       - General disclaimers → Disclaimer

    5. TOTALS - Find the final amounts:
       - Look for "Subtotal", "Tax", "Total", "Grand Total"
       - grandTotal should be the final amount due

    6. CONTRACTOR INFO - Usually at top of document:
       - Company name, address, phone, email

    7. CLIENT INFO - Usually below contractor:
       - Customer/client name, project name or job site address

    COMMON UNITS: SF (Square Feet), LF (Linear Feet), EA (Each), HR (Hour), LS (Lump Sum)

    {{ ctx.output_format }}
  "#
}

// Simplified version for fallback or quick extraction
function ExtractEstimateSimple(pdfText: string) -> ParsedEstimate {
  client CloudflareLlama

  prompt #"
    Extract data from this construction estimate.

    {{ pdfText }}

    Rules:
    - Extract ALL line items with prices
    - Keep EXACT descriptions
    - Numbers only (no $ or commas)

    {{ ctx.output_format }}
  "#
}
```

### Classification Function (baml_src/classification.baml)

```baml
// baml_src/classification.baml

// Input for classification
class ClassificationInput {
  id string
  description string
  sectionHeader string?
  unit string?
}

function ClassifyLineItems(
  items: ClassificationInput[],
  costCodeReference: string
) -> ClassificationResult[] {
  client CloudflareLlama

  prompt #"
    You are a construction cost classification expert. Assign cost codes to these line items.

    AVAILABLE COST CODES:
    {{ costCodeReference }}

    LINE ITEMS TO CLASSIFY:
    {% for item in items %}
    [{{ item.id }}] {{ item.description }}{% if item.sectionHeader %} [Section: {{ item.sectionHeader }}]{% endif %}{% if item.unit %} [Unit: {{ item.unit }}]{% endif %}
    {% endfor %}

    CLASSIFICATION RULES:

    1. CODE SELECTION
       - Use the MOST SPECIFIC code that applies (level 2 over level 1)
       - Section header is strong context for division selection
       - When uncertain, use the parent division code

    2. COST TYPE CLASSIFICATION (Level 2)
       - Labor codes (_L): Installation labor, crew hours, worker costs
       - Material codes (_M): Products, supplies, equipment purchases
       - Subcontract codes (_S): Third-party work, specialty contractors

    3. CONFIDENCE LEVELS
       - 0.90-1.00: Exact match, clear category
       - 0.70-0.89: Good match, minor ambiguity
       - 0.50-0.69: Reasonable match, some uncertainty
       - Below 0.50: Uncertain, use broader category

    4. CONTEXT CLUES
       - Units like "HR" suggest labor (_L)
       - "Subcontract" or "Sub" in description → _S codes
       - Material names (lumber, pipe, wire) → _M codes
       - "Install" or "labor" → _L codes

    5. ALTERNATES
       - Provide 1-2 alternate codes when classification is uncertain

    DIVISION MAPPINGS:
    00: Demolition (tear out, remove, demo, abatement)
    01: Sitework (grading, excavation, landscaping)
    02: Foundation (footing, slab, concrete foundation)
    03: Framing (studs, joists, rafters, trusses)
    04: Roofing (shingles, flashing, gutters)
    05: Windows & Doors
    06: Siding & Trim (siding, stucco, exterior trim)
    07: Electrical (wiring, panel, outlets, lighting)
    08: HVAC (furnace, AC, ductwork)
    09: Plumbing (pipes, fixtures, water heater)
    10: Insulation
    11: Carpentry (finish carpentry, trim, molding)
    12: Casework (cabinets, countertops)
    13: Drywall & Finishes (drywall, paint, tile, flooring)
    14: Specialties (appliances, fireplace)
    15: General Conditions (supervision, permits, cleanup)

    {{ ctx.output_format }}
  "#
}

// Single item classification for reclassification UI
function ClassifySingleItem(
  description: string,
  sectionHeader: string?,
  costCodeReference: string
) -> ClassificationResult {
  client CloudflareLlama

  prompt #"
    Classify this construction line item to a cost code.

    AVAILABLE CODES:
    {{ costCodeReference }}

    LINE ITEM: {{ description }}
    {% if sectionHeader %}SECTION: {{ sectionHeader }}{% endif %}

    Select the best matching code with confidence score and up to 2 alternates.

    {{ ctx.output_format }}
  "#
}
```

### Test Cases (baml_src/tests/extraction_tests.baml)

```baml
// baml_src/tests/extraction_tests.baml

test SimpleEstimate {
  functions [ExtractEstimate]
  args {
    filename "kitchen_estimate.pdf"
    pdfText #"
      ABC CONSTRUCTION
      123 Main St, Anytown, USA
      (555) 123-4567

      Customer: John Smith
      Project: Kitchen Remodel

      DEMOLITION
      Remove cabinets                    $500.00
      Dispose debris                     $300.00

      CABINETS
      Base cabinets 12 LF @ $150        $1,800.00
      Upper cabinets 8 LF @ $125        $1,000.00

      Subtotal:                         $3,600.00
      Tax:                                $288.00
      Total:                            $3,888.00

      Exclusions:
      - Permits not included
      - Electrical work by others
    "#
  }
}

test TableFormatEstimate {
  functions [ExtractEstimate]
  args {
    filename "table_estimate.pdf"
    pdfText #"
      ACME BUILDERS | License #12345
      456 Oak Ave, Builder City, ST 12345
      Phone: (555) 987-6543 | Email: info@acme.com

      PROPOSAL FOR: Sarah Johnson
      PROJECT: Bathroom Renovation - 123 Elm St

      ================================================================
      Description              Qty    Unit   Unit Price    Total
      ================================================================
      PLUMBING
      Remove existing fixtures  1     LS                   $450.00
      New toilet               1     EA     $350.00       $350.00
      New vanity w/ sink       1     EA     $800.00       $800.00
      Plumbing labor           8     HR     $85.00        $680.00

      TILE
      Floor tile               45    SF     $12.00        $540.00
      Tile installation labor  45    SF     $8.00         $360.00
      ================================================================
                                           SUBTOTAL:     $3,180.00
                                           TAX (8%):       $254.40
                                           TOTAL:        $3,434.40

      Notes:
      - Price valid for 30 days
      - 50% deposit required to schedule
    "#
  }
}
```

### Updated Services

#### Estimate Parser (packages/backend/src/services/estimate-parser.ts)

```typescript
// packages/backend/src/services/estimate-parser.ts
import { b } from '../../baml_client';
import type { ParsedEstimate } from '../../baml_client/types';

export class EstimateParser {
  private readonly maxInputTokens = 12000;

  /**
   * Parse raw PDF text into structured estimate data using BAML
   */
  async parse(rawText: string, filename: string): Promise<ParsedEstimate> {
    // Truncate if necessary
    const truncatedText = this.truncateText(rawText, this.maxInputTokens);

    try {
      // BAML handles JSON parsing, validation, and retry automatically
      const result = await b.ExtractEstimate(truncatedText, filename);
      return this.postProcess(result);
    } catch (error) {
      console.error('BAML extraction failed, trying simple extraction:', error);

      // Fallback to simpler extraction
      try {
        const result = await b.ExtractEstimateSimple(truncatedText);
        return this.postProcess(result);
      } catch (fallbackError) {
        console.error('Simple extraction also failed:', fallbackError);
        throw new EstimateParseError(
          'Failed to extract estimate data',
          error instanceof Error ? error.message : String(error)
        );
      }
    }
  }

  /**
   * Truncate text to approximate token limit
   */
  private truncateText(text: string, maxTokens: number): string {
    const maxChars = maxTokens * 4;

    if (text.length <= maxChars) {
      return text;
    }

    const keepChars = maxChars - 100;
    const halfKeep = Math.floor(keepChars / 2);

    return (
      text.slice(0, halfKeep) +
      '\n\n[... content truncated ...]\n\n' +
      text.slice(-halfKeep)
    );
  }

  /**
   * Post-process BAML result (calculate missing totals, etc.)
   */
  private postProcess(result: ParsedEstimate): ParsedEstimate {
    // Calculate grand total if not provided
    if (!result.documentTotals.grandTotal && result.lineItems.length > 0) {
      result.documentTotals.grandTotal = result.lineItems.reduce(
        (sum, item) => sum + item.totalPrice,
        0
      );
    }

    return result;
  }
}

export class EstimateParseError extends Error {
  constructor(
    message: string,
    public readonly details?: string
  ) {
    super(message);
    this.name = 'EstimateParseError';
  }
}
```

#### Classifier (packages/backend/src/services/classifier.ts)

```typescript
// packages/backend/src/services/classifier.ts
import { b } from '../../baml_client';
import type { ClassificationResult, ClassificationInput } from '../../baml_client/types';
import type { CostCode } from '@estimate-compare/shared';

export class Classifier {
  private readonly batchSize = 15;

  /**
   * Classify multiple line items to cost codes using BAML
   */
  async classifyLineItems(
    items: ClassificationInput[],
    costCodes: CostCode[]
  ): Promise<Map<string, ClassificationResult>> {
    const results = new Map<string, ClassificationResult>();

    if (items.length === 0) {
      return results;
    }

    const codeReference = this.buildCodeReference(costCodes);
    const validCodes = new Set(costCodes.map(c => c.code));

    // Process in batches
    for (let i = 0; i < items.length; i += this.batchSize) {
      const batch = items.slice(i, i + this.batchSize);

      try {
        // BAML handles JSON parsing and validation
        const batchResults = await b.ClassifyLineItems(batch, codeReference);

        for (const result of batchResults) {
          // Validate code exists
          if (validCodes.has(result.code)) {
            results.set(result.id, result);
          } else {
            results.set(result.id, this.createDefaultClassification(
              batch.find(item => item.id === result.id)!,
              costCodes
            ));
          }
        }
      } catch (error) {
        console.error(`Batch classification failed:`, error);

        // Fall back to default classification
        for (const item of batch) {
          results.set(item.id, this.createDefaultClassification(item, costCodes));
        }
      }
    }

    return results;
  }

  /**
   * Classify a single item (for reclassification UI)
   */
  async classifySingle(
    description: string,
    sectionHeader: string | null,
    costCodes: CostCode[]
  ): Promise<ClassificationResult> {
    const codeReference = this.buildCodeReference(costCodes);

    try {
      return await b.ClassifySingleItem(description, sectionHeader, codeReference);
    } catch (error) {
      console.error('Single classification failed:', error);
      return this.createDefaultClassification(
        { id: 'single', description, sectionHeader },
        costCodes
      );
    }
  }

  /**
   * Build cost code reference string for prompt
   */
  private buildCodeReference(costCodes: CostCode[]): string {
    const level1Codes = costCodes.filter(c => c.level === 1);
    const lines: string[] = [];

    for (const l1 of level1Codes.sort((a, b) => a.sortOrder - b.sortOrder)) {
      lines.push(`${l1.code}: ${l1.label}`);

      const children = costCodes
        .filter(c => c.parentCode === l1.code)
        .sort((a, b) => a.sortOrder - b.sortOrder);

      for (const child of children) {
        lines.push(`  ${child.code}: ${child.label}`);
      }
    }

    return lines.join('\n');
  }

  /**
   * Create default classification when LLM fails
   */
  private createDefaultClassification(
    item: ClassificationInput,
    costCodes: CostCode[]
  ): ClassificationResult {
    const inferredCode = this.inferFromSection(item.sectionHeader, costCodes);

    return {
      id: item.id,
      code: inferredCode || '15',
      confidence: inferredCode ? 0.4 : 0.2,
      alternates: [],
    };
  }

  /**
   * Infer code from section header
   */
  private inferFromSection(
    section: string | null,
    costCodes: CostCode[]
  ): string | null {
    if (!section) return null;

    const sectionLower = section.toLowerCase();
    const level1Codes = costCodes.filter(c => c.level === 1);

    for (const code of level1Codes) {
      if (sectionLower.includes(code.label.toLowerCase())) {
        return code.code;
      }
    }

    return null;
  }
}
```

## Development Workflow

### Initial Setup

```bash
# Install BAML CLI
npm install -g @boundaryml/baml

# Or add as dev dependency
pnpm add -D @boundaryml/baml

# Initialize BAML in project (creates baml_src/ and config)
baml init

# Install VS Code extension for playground
# Search "BAML" in VS Code extensions
```

### Development Commands

```bash
# Generate TypeScript client from .baml files
baml generate

# Watch mode - regenerate on changes
baml generate --watch

# Validate .baml files without generating
baml check

# Run tests in playground
baml test
```

### VS Code Playground

The BAML VS Code extension provides:

1. **Syntax highlighting** for `.baml` files
2. **Inline playground** - Run functions with test data directly in editor
3. **Output preview** - See parsed results alongside prompts
4. **Error highlighting** - Catch schema issues before runtime

### Iteration Workflow

1. **Edit prompt** in `.baml` file
2. **Test in playground** with sample estimate text
3. **Review output** - check extraction accuracy
4. **Refine prompt** based on results
5. **Run `baml generate`** to update TypeScript client
6. **Test in application** with real PDFs
7. **Commit changes** - `.baml` files are version controlled

## Migration Plan

### Phase 1: Setup (Day 1)

- [ ] Install BAML CLI and VS Code extension
- [ ] Create `baml.config.yaml`
- [ ] Create `baml_src/` directory structure
- [ ] Configure Cloudflare client in `clients.baml`
- [ ] Add `baml generate` to build scripts
- [ ] Update `.gitignore` for `baml_client/`

### Phase 2: Types Migration (Day 1-2)

- [ ] Define types in `types.baml`
- [ ] Map existing TypeScript interfaces to BAML classes
- [ ] Verify generated TypeScript matches current types
- [ ] Update shared package to re-export BAML types

### Phase 3: Extraction Migration (Day 2-3)

- [ ] Create `extraction.baml` with ExtractEstimate function
- [ ] Port EXTRACTION_SYSTEM_PROMPT to BAML format
- [ ] Create test cases with sample estimates
- [ ] Test in VS Code playground
- [ ] Update `estimate-parser.ts` to use BAML client
- [ ] Remove manual JSON parsing code
- [ ] Test with real PDF uploads

### Phase 4: Classification Migration (Day 3-4)

- [ ] Create `classification.baml` with ClassifyLineItems function
- [ ] Port CLASSIFICATION_SYSTEM_PROMPT to BAML format
- [ ] Create test cases for classification
- [ ] Test in VS Code playground
- [ ] Update `classifier.ts` to use BAML client
- [ ] Test batch classification
- [ ] Test single item reclassification

### Phase 5: Cleanup & Documentation (Day 4-5)

- [ ] Remove old `prompts/` directory
- [ ] Remove manual JSON extraction utilities
- [ ] Update module documentation
- [ ] Document BAML workflow for team
- [ ] Add BAML commands to package.json scripts
- [ ] Update CI/CD to include `baml generate`

## Configuration for Cloudflare Workers

### Environment Variables

```toml
# wrangler.toml
[vars]
# CF_ACCOUNT_ID is automatically available in Workers

[secrets]
# Add via: wrangler secret put CF_API_TOKEN
# CF_API_TOKEN = "your-api-token"

# Optional: for fallback to other providers
# OPENAI_API_KEY = "sk-..."
# ANTHROPIC_API_KEY = "sk-ant-..."
```

### Build Integration

```json
// package.json
{
  "scripts": {
    "baml:generate": "baml generate",
    "baml:watch": "baml generate --watch",
    "baml:check": "baml check",
    "prebuild": "pnpm baml:generate",
    "dev": "concurrently \"pnpm baml:watch\" \"wrangler dev\""
  }
}
```

## Testing Strategy

### Unit Tests

```typescript
// packages/backend/src/services/__tests__/estimate-parser.test.ts
import { describe, it, expect, vi } from 'vitest';
import { EstimateParser } from '../estimate-parser';

// Mock BAML client
vi.mock('../../../baml_client', () => ({
  b: {
    ExtractEstimate: vi.fn(),
    ExtractEstimateSimple: vi.fn(),
  },
}));

describe('EstimateParser', () => {
  it('should extract estimate data', async () => {
    const parser = new EstimateParser();
    const result = await parser.parse(samplePdfText, 'test.pdf');

    expect(result.lineItems.length).toBeGreaterThan(0);
    expect(result.documentTotals.grandTotal).toBeGreaterThan(0);
  });
});
```

### BAML Playground Tests

Create test cases in `baml_src/tests/` that can be run via:
- VS Code playground (interactive)
- `baml test` command (CI/CD)

### Integration Tests

Test full flow with real PDF files:

```typescript
describe('Extraction Integration', () => {
  it('should extract from real kitchen estimate PDF', async () => {
    const pdfText = await extractPdfText('fixtures/kitchen-estimate.pdf');
    const result = await parser.parse(pdfText, 'kitchen-estimate.pdf');

    // Verify expected line items are extracted
    expect(result.lineItems.some(i =>
      i.description.toLowerCase().includes('cabinet')
    )).toBe(true);
  });
});
```

## Metrics & Monitoring

### Extraction Quality Metrics

```typescript
interface ExtractionMetrics {
  totalItems: number;
  itemsWithSection: number;
  itemsWithQuantity: number;
  avgDescriptionLength: number;
  parseTimeMs: number;
  modelUsed: string;
  fallbackUsed: boolean;
}

function calculateExtractionMetrics(
  result: ParsedEstimate,
  parseTimeMs: number,
  modelUsed: string,
  fallbackUsed: boolean
): ExtractionMetrics {
  return {
    totalItems: result.lineItems.length,
    itemsWithSection: result.lineItems.filter(i => i.sectionHeader).length,
    itemsWithQuantity: result.lineItems.filter(i => i.quantity).length,
    avgDescriptionLength: result.lineItems.reduce(
      (sum, i) => sum + i.description.length, 0
    ) / result.lineItems.length,
    parseTimeMs,
    modelUsed,
    fallbackUsed,
  };
}
```

### Logging

```typescript
// Log metrics for analysis
console.log('Extraction metrics:', JSON.stringify({
  sessionId,
  estimateId,
  filename,
  ...metrics,
  timestamp: new Date().toISOString(),
}));
```

## Future Enhancements

### A/B Testing Prompts

```baml
// Define prompt variants
function ExtractEstimateV2(pdfText: string, filename: string) -> ParsedEstimate {
  client ExtractionClient
  prompt #"
    // New prompt variant with different instructions
    ...
  "#
}
```

```typescript
// Route to variant based on session ID
const extractFn = sessionId.hashCode() % 2 === 0
  ? b.ExtractEstimate
  : b.ExtractEstimateV2;
```

### Few-Shot Learning

```baml
function ExtractEstimateWithExamples(
  pdfText: string,
  filename: string,
  examples: ExtractionExample[]
) -> ParsedEstimate {
  client ExtractionClient
  prompt #"
    Here are examples of correctly extracted estimates:

    {% for example in examples %}
    INPUT:
    {{ example.input }}

    OUTPUT:
    {{ example.output }}
    ---
    {% endfor %}

    Now extract from this document:
    {{ pdfText }}

    {{ ctx.output_format }}
  "#
}
```

### Streaming Extraction

BAML supports streaming for real-time UI updates:

```typescript
// Stream partial results as they're generated
const stream = await b.stream.ExtractEstimate(pdfText, filename);

for await (const partial of stream) {
  // Update UI with partial results
  updateExtractionProgress(partial);
}

const final = await stream.finalResponse();
```

## Todo List

### Setup

- [ ] Install BAML CLI (`npm install -g @boundaryml/baml`)
- [ ] Install VS Code BAML extension
- [ ] Create `baml.config.yaml`
- [ ] Create `baml_src/` directory
- [ ] Add BAML scripts to package.json
- [ ] Update `.gitignore` for `baml_client/`

### Implementation

- [ ] Create `clients.baml` with Cloudflare client
- [ ] Create `types.baml` with all type definitions
- [ ] Create `extraction.baml` with ExtractEstimate function
- [ ] Create `classification.baml` with ClassifyLineItems function
- [ ] Create test cases in `baml_src/tests/`
- [ ] Run `baml generate` to create TypeScript client
- [ ] Update `estimate-parser.ts` to use BAML
- [ ] Update `classifier.ts` to use BAML
- [ ] Remove old `prompts/` directory
- [ ] Remove manual JSON parsing code

### Testing

- [ ] Test extraction in VS Code playground
- [ ] Test classification in VS Code playground
- [ ] Run unit tests with mocked BAML client
- [ ] Test integration with real PDF uploads
- [ ] Verify Cloudflare Workers AI compatibility
- [ ] Test fallback to OpenAI (if configured)

### Documentation

- [ ] Update README.md with BAML information
- [ ] Document BAML development workflow
- [ ] Update module 05 and 06 to reference BAML
- [ ] Add troubleshooting guide

## Verification Checklist

- [ ] BAML generates TypeScript client successfully
- [ ] Types match existing interfaces
- [ ] Extraction produces same quality results as before
- [ ] Classification accuracy is maintained or improved
- [ ] VS Code playground works for testing
- [ ] Cloudflare Workers AI client works in production
- [ ] Fallback strategy works when primary fails
- [ ] Build pipeline includes `baml generate`
- [ ] Error handling is robust

## Time Estimate

| Task | Estimate |
|------|----------|
| Setup & configuration | 2 hours |
| Type definitions | 1 hour |
| Extraction migration | 3 hours |
| Classification migration | 2 hours |
| Testing & refinement | 3 hours |
| Documentation | 1 hour |
| **Total** | **~12 hours** |

## References

- [BAML Documentation](https://docs.boundaryml.com/home)
- [BAML GitHub Repository](https://github.com/BoundaryML/baml)
- [BAML VS Code Extension](https://marketplace.visualstudio.com/items?itemName=Boundary.baml-extension)
- [Schema-Aligned Parsing Blog Post](https://boundaryml.com/blog/structured-output-from-llms)
