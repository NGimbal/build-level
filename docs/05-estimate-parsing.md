# Module 05: Estimate Parsing

## Overview

LLM-based parsing service that transforms raw PDF text into structured estimate data. Uses [BAML](https://github.com/BoundaryML/baml) for type-safe prompt engineering with Workers AI (Llama 3.1).

## Goals

- Parse unstructured PDF text into structured JSON
- Extract all line items with descriptions and amounts
- Identify contractor and client information
- Capture notes, exclusions, and disclaimers
- Handle various estimate formats robustly

## Why BAML

BAML (Boundary AI Markup Language) provides:

| Feature | Benefit |
|---------|---------|
| Schema-Aligned Parsing | Handles markdown blocks, extra text, minor JSON errors automatically |
| Type Generation | Compiles `.baml` files to TypeScript interfaces |
| VS Code Playground | Test prompts interactively without deploying |
| Multi-Model Support | Swap between Llama/GPT/Claude with one line change |
| Version Control | Prompts in `.baml` files are easy to diff and review |

## File Structure

```
build-level/
├── baml_src/                    # BAML source files
│   ├── clients.baml             # LLM client configurations
│   ├── types.baml               # Shared type definitions
│   ├── extraction.baml          # Estimate extraction function
│   └── tests/
│       └── extraction_tests.baml
├── baml_client/                 # Generated TypeScript (gitignored)
├── packages/backend/src/services/
│   └── estimate-parser.ts       # Parser service using BAML client
└── baml.config.yaml
```

## BAML Setup

### Configuration (baml.config.yaml)

```yaml
version: 1
output_dir: ./baml_client

generators:
  - language: typescript
    output_dir: ./baml_client

default_client: cloudflare-llama
```

### Client Configuration (baml_src/clients.baml)

```baml
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

// Fallback: OpenAI for higher accuracy when needed
client<llm> OpenAIGPT4 {
  provider openai
  options {
    model "gpt-4-turbo-preview"
    api_key env.OPENAI_API_KEY
  }
}

// Extraction client with fallback strategy
client<llm> ExtractionClient {
  strategy fallback
  options {
    clients [CloudflareLlama, OpenAIGPT4]
  }
}
```

## Type Definitions (baml_src/types.baml)

```baml
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
```

## Extraction Function (baml_src/extraction.baml)

```baml
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

// Simplified version for fallback
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

## Core Implementation

### Estimate Parser Service (estimate-parser.ts)

```typescript
import { b } from '../../baml_client';
import type { ParsedEstimate } from '../../baml_client/types';

export class EstimateParser {
  private readonly maxInputTokens = 12000;

  /**
   * Parse raw PDF text into structured estimate data
   */
  async parse(rawText: string, filename: string): Promise<ParsedEstimate> {
    const truncatedText = this.truncateText(rawText, this.maxInputTokens);

    try {
      // BAML handles JSON parsing, validation, and retry automatically
      const result = await b.ExtractEstimate(truncatedText, filename);
      return this.postProcess(result);
    } catch (error) {
      console.error('Extraction failed, trying simple extraction:', error);

      // Fallback to simpler extraction
      try {
        const result = await b.ExtractEstimateSimple(truncatedText);
        return this.postProcess(result);
      } catch (fallbackError) {
        console.error('Simple extraction also failed:', fallbackError);

        // Last resort: basic regex extraction
        return this.extractBasicLineItems(rawText);
      }
    }
  }

  /**
   * Truncate text to approximate token limit
   * (rough estimate: 1 token ≈ 4 characters)
   */
  private truncateText(text: string, maxTokens: number): string {
    const maxChars = maxTokens * 4;

    if (text.length <= maxChars) {
      return text;
    }

    // Keep beginning and end, truncate middle
    const keepChars = maxChars - 100;
    const halfKeep = Math.floor(keepChars / 2);

    return (
      text.slice(0, halfKeep) +
      '\n\n[... content truncated ...]\n\n' +
      text.slice(-halfKeep)
    );
  }

  /**
   * Post-process BAML result
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

  /**
   * Basic regex extraction as last resort fallback
   */
  private extractBasicLineItems(text: string): ParsedEstimate {
    const items: ParsedEstimate['lineItems'] = [];
    const lines = text.split('\n');
    const amountPattern = /\$?([\d,]+\.?\d*)\s*$/;

    for (const line of lines) {
      const match = line.match(amountPattern);
      if (match) {
        const amount = parseFloat(match[1].replace(/,/g, ''));
        if (amount > 0 && amount < 10000000) {
          const description = line.slice(0, match.index).trim();
          if (description.length > 3) {
            items.push({
              description,
              sectionHeader: null,
              quantity: null,
              unit: null,
              unitPrice: null,
              totalPrice: amount,
            });
          }
        }
      }
    }

    return {
      contractor: { name: null, address: null, phone: null, email: null },
      client: null,
      lineItems: items,
      notes: [],
      documentTotals: {
        subtotal: null,
        tax: null,
        grandTotal: items.reduce((s, i) => s + i.totalPrice, 0),
      },
    };
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

## Test Cases (baml_src/tests/extraction_tests.baml)

```baml
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

## Usage Example

```typescript
// In session processing

import { EstimateParser } from '../services/estimate-parser';

async function parseEstimate(
  rawText: string,
  filename: string
): Promise<ParsedEstimate> {
  const parser = new EstimateParser();
  return parser.parse(rawText, filename);
}
```

## Development Workflow

### Setup

```bash
# Install BAML CLI
pnpm add -D @boundaryml/baml

# Generate TypeScript client
pnpm baml generate

# Add to package.json scripts
# "baml:generate": "baml generate",
# "baml:watch": "baml generate --watch",
# "prebuild": "pnpm baml:generate"
```

### Iterating on Prompts

1. Edit `extraction.baml` in VS Code
2. Use BAML playground to test with sample estimate text
3. Review extracted output
4. Refine prompt based on results
5. Run `baml generate` to update TypeScript client
6. Test in application with real PDFs

## Todo List

### BAML Setup

- [ ] Install BAML CLI (`pnpm add -D @boundaryml/baml`)
- [ ] Install VS Code BAML extension
- [ ] Create `baml.config.yaml`
- [ ] Create `baml_src/` directory
- [ ] Add BAML scripts to package.json
- [ ] Add `baml_client/` to `.gitignore`

### Type Definitions

- [ ] Create `types.baml` with all extraction types
- [ ] Run `baml generate` to verify TypeScript output
- [ ] Verify generated types match expected interfaces

### Extraction Function

- [ ] Create `clients.baml` with Cloudflare client config
- [ ] Create `extraction.baml` with ExtractEstimate function
- [ ] Create `ExtractEstimateSimple` fallback function
- [ ] Test extraction in VS Code playground
- [ ] Refine prompt based on test results

### Parser Service

- [ ] Create estimate-parser.ts using BAML client
- [ ] Implement truncateText for long documents
- [ ] Implement postProcess for calculated totals
- [ ] Implement extractBasicLineItems fallback
- [ ] Create EstimateParseError class

### Testing

- [ ] Create test cases in `baml_src/tests/`
- [ ] Test with simple one-page estimate
- [ ] Test with table-formatted estimate
- [ ] Test with multi-section estimate
- [ ] Test with poorly structured estimate
- [ ] Test fallback extraction
- [ ] Verify Cloudflare Workers AI compatibility

## Verification Checklist

- [ ] BAML generates TypeScript client successfully
- [ ] Parser extracts contractor info correctly
- [ ] Parser extracts client info correctly
- [ ] All line items with prices are extracted
- [ ] Line item descriptions are preserved exactly
- [ ] Section headers are assigned correctly
- [ ] Numbers are parsed correctly (no $ or commas)
- [ ] Notes and exclusions are categorized
- [ ] Document totals are extracted or calculated
- [ ] Fallback extraction works when LLM fails
- [ ] VS Code playground works for testing

## Notes

### Workers AI Limitations

- Model: `@cf/meta/llama-3.1-8b-instruct`
- Max input: ~16K tokens (practical limit ~12K)
- Max output: 4K tokens
- Response time: 2-10 seconds typically

### Prompt Engineering Tips

1. **Low temperature** - BAML uses 0.1 for consistent output
2. **Explicit format** - `{{ ctx.output_format }}` injects schema
3. **Negative instructions** - "Do NOT paraphrase" helps
4. **Section context** - Include section headers in extraction

### Sample Input → Output

**Input:**
```
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

Subtotal:                         $2,600.00
Tax:                                $208.00
Total:                            $2,808.00
```

**Output:**
```json
{
  "contractor": {
    "name": "ABC CONSTRUCTION",
    "address": "123 Main St, Anytown, USA",
    "phone": "(555) 123-4567",
    "email": null
  },
  "client": {
    "name": "John Smith",
    "projectName": "Kitchen Remodel"
  },
  "lineItems": [
    {"description": "Remove cabinets", "sectionHeader": "DEMOLITION", "quantity": null, "unit": null, "unitPrice": null, "totalPrice": 500.00},
    {"description": "Dispose debris", "sectionHeader": "DEMOLITION", "quantity": null, "unit": null, "unitPrice": null, "totalPrice": 300.00},
    {"description": "Base cabinets", "sectionHeader": "CABINETS", "quantity": 12, "unit": "LF", "unitPrice": 150, "totalPrice": 1800.00}
  ],
  "notes": [],
  "documentTotals": {
    "subtotal": 2600.00,
    "tax": 208.00,
    "grandTotal": 2808.00
  }
}
```

## Time Estimate

| Task | Estimate |
|------|----------|
| BAML setup | 1 hour |
| Type definitions | 30 min |
| Extraction function | 1 hour |
| Parser service | 1.5 hours |
| Testing & refinement | 2 hours |
| **Total** | **~6 hours** |
