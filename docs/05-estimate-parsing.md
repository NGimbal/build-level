# Module 05: Estimate Parsing

## Overview

LLM-based parsing service that transforms raw PDF text into structured estimate data. Uses Workers AI (Llama 3.1) to extract contractor info, line items, notes, and totals.

## Goals

- Parse unstructured PDF text into structured JSON
- Extract all line items with descriptions and amounts
- Identify contractor and client information
- Capture notes, exclusions, and disclaimers
- Handle various estimate formats robustly

## File Structure

```
packages/backend/src/services/
├── estimate-parser.ts    # Main parsing service
└── prompts/
    └── extraction.ts     # Extraction prompt templates
```

## Core Implementation

### Estimate Parser Service (estimate-parser.ts)

```typescript
import type { Ai } from '@cloudflare/workers-types';

export interface ParsedContractor {
  name: string | null;
  address: string | null;
  phone: string | null;
  email: string | null;
}

export interface ParsedClient {
  name: string | null;
  projectName: string | null;
}

export interface ParsedLineItem {
  description: string;
  sectionHeader: string | null;
  quantity: number | null;
  unit: string | null;
  unitPrice: number | null;
  totalPrice: number;
}

export interface ParsedNote {
  noteType: 'note' | 'exclusion' | 'disclaimer' | 'term';
  content: string;
}

export interface ParsedDocumentTotals {
  subtotal: number | null;
  tax: number | null;
  grandTotal: number;
}

export interface ParsedEstimate {
  contractor: ParsedContractor;
  client: ParsedClient | null;
  lineItems: ParsedLineItem[];
  notes: ParsedNote[];
  documentTotals: ParsedDocumentTotals;
}

export class EstimateParser {
  private readonly model = '@cf/meta/llama-3.1-8b-instruct';
  private readonly maxInputTokens = 12000;  // Leave room for response
  
  constructor(private ai: Ai) {}

  /**
   * Parse raw PDF text into structured estimate data
   */
  async parse(rawText: string, filename: string): Promise<ParsedEstimate> {
    // Truncate if necessary
    const truncatedText = this.truncateText(rawText, this.maxInputTokens);
    
    const prompt = this.buildPrompt(truncatedText, filename);
    
    const response = await this.ai.run(this.model, {
      messages: [
        { role: 'system', content: EXTRACTION_SYSTEM_PROMPT },
        { role: 'user', content: prompt },
      ],
      max_tokens: 4000,
      temperature: 0.1,  // Low temperature for consistent output
    });

    const responseText = this.extractResponseText(response);
    
    return this.parseResponse(responseText, rawText);
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
    const keepChars = maxChars - 100;  // Room for truncation notice
    const halfKeep = Math.floor(keepChars / 2);
    
    return (
      text.slice(0, halfKeep) +
      '\n\n[... content truncated ...]\n\n' +
      text.slice(-halfKeep)
    );
  }

  /**
   * Build the user prompt for extraction
   */
  private buildPrompt(text: string, filename: string): string {
    return `Parse this construction estimate document.

FILENAME: ${filename}

DOCUMENT TEXT:
${text}

Extract all information and return ONLY valid JSON matching the schema in the system prompt. No other text.`;
  }

  /**
   * Extract text from AI response object
   */
  private extractResponseText(response: unknown): string {
    if (typeof response === 'string') {
      return response;
    }
    
    if (response && typeof response === 'object' && 'response' in response) {
      return String((response as { response: unknown }).response);
    }
    
    throw new EstimateParseError('Unexpected AI response format');
  }

  /**
   * Parse and validate the JSON response
   */
  private parseResponse(responseText: string, originalText: string): ParsedEstimate {
    // Extract JSON from response (handle markdown code blocks)
    const json = this.extractJSON(responseText);
    
    try {
      const parsed = JSON.parse(json);
      return this.validateAndNormalize(parsed, originalText);
    } catch (error) {
      throw new EstimateParseError(
        `Failed to parse AI response as JSON: ${error instanceof Error ? error.message : 'Unknown error'}`,
        responseText
      );
    }
  }

  /**
   * Extract JSON from response text (handles markdown blocks)
   */
  private extractJSON(text: string): string {
    // Try to find JSON in markdown code block
    const codeBlockMatch = text.match(/```(?:json)?\s*([\s\S]*?)```/);
    if (codeBlockMatch) {
      return codeBlockMatch[1].trim();
    }
    
    // Try to find raw JSON object
    const objectMatch = text.match(/\{[\s\S]*\}/);
    if (objectMatch) {
      return objectMatch[0];
    }
    
    // Return as-is if nothing found
    return text.trim();
  }

  /**
   * Validate and normalize the parsed data
   */
  private validateAndNormalize(data: unknown, originalText: string): ParsedEstimate {
    if (!data || typeof data !== 'object') {
      throw new EstimateParseError('Response is not an object');
    }
    
    const obj = data as Record<string, unknown>;
    
    // Normalize contractor
    const contractor: ParsedContractor = {
      name: this.normalizeString(obj.contractor, 'name'),
      address: this.normalizeString(obj.contractor, 'address'),
      phone: this.normalizeString(obj.contractor, 'phone'),
      email: this.normalizeString(obj.contractor, 'email'),
    };
    
    // Normalize client
    let client: ParsedClient | null = null;
    if (obj.client && typeof obj.client === 'object') {
      const c = obj.client as Record<string, unknown>;
      if (c.name || c.projectName) {
        client = {
          name: this.normalizeString(c, 'name'),
          projectName: this.normalizeString(c, 'projectName'),
        };
      }
    }
    
    // Normalize line items
    const lineItems = this.normalizeLineItems(obj.lineItems);
    
    // Normalize notes
    const notes = this.normalizeNotes(obj.notes);
    
    // Normalize document totals
    const documentTotals = this.normalizeDocumentTotals(obj.documentTotals, lineItems);
    
    return {
      contractor,
      client,
      lineItems,
      notes,
      documentTotals,
    };
  }

  /**
   * Safely extract string from nested object
   */
  private normalizeString(
    parent: unknown,
    key: string
  ): string | null {
    if (!parent || typeof parent !== 'object') {
      return null;
    }
    
    const value = (parent as Record<string, unknown>)[key];
    
    if (value === null || value === undefined) {
      return null;
    }
    
    const str = String(value).trim();
    return str.length > 0 ? str : null;
  }

  /**
   * Normalize line items array
   */
  private normalizeLineItems(items: unknown): ParsedLineItem[] {
    if (!Array.isArray(items)) {
      return [];
    }
    
    const result: ParsedLineItem[] = [];
    
    for (const item of items) {
      if (!item || typeof item !== 'object') {
        continue;
      }
      
      const obj = item as Record<string, unknown>;
      
      // Description is required
      const description = this.normalizeString(obj, 'description');
      if (!description) {
        continue;
      }
      
      // Total price is required
      const totalPrice = this.normalizeNumber(obj.totalPrice);
      if (totalPrice === null || totalPrice <= 0) {
        continue;
      }
      
      result.push({
        description,
        sectionHeader: this.normalizeString(obj, 'sectionHeader') || 
                       this.normalizeString(obj, 'section') ||
                       this.normalizeString(obj, 'section_header'),
        quantity: this.normalizeNumber(obj.quantity),
        unit: this.normalizeString(obj, 'unit'),
        unitPrice: this.normalizeNumber(obj.unitPrice) || 
                   this.normalizeNumber(obj.unit_price),
        totalPrice,
      });
    }
    
    return result;
  }

  /**
   * Normalize number value
   */
  private normalizeNumber(value: unknown): number | null {
    if (value === null || value === undefined) {
      return null;
    }
    
    if (typeof value === 'number') {
      return isNaN(value) ? null : value;
    }
    
    if (typeof value === 'string') {
      // Remove currency symbols, commas, spaces
      const cleaned = value.replace(/[$,\s]/g, '');
      const parsed = parseFloat(cleaned);
      return isNaN(parsed) ? null : parsed;
    }
    
    return null;
  }

  /**
   * Normalize notes array
   */
  private normalizeNotes(notes: unknown): ParsedNote[] {
    if (!Array.isArray(notes)) {
      return [];
    }
    
    const validTypes = ['note', 'exclusion', 'disclaimer', 'term'];
    const result: ParsedNote[] = [];
    
    for (const note of notes) {
      if (!note || typeof note !== 'object') {
        continue;
      }
      
      const obj = note as Record<string, unknown>;
      const content = this.normalizeString(obj, 'content');
      
      if (!content) {
        continue;
      }
      
      let noteType = this.normalizeString(obj, 'noteType') ||
                     this.normalizeString(obj, 'type') ||
                     'note';
      
      if (!validTypes.includes(noteType)) {
        noteType = 'note';
      }
      
      result.push({
        noteType: noteType as ParsedNote['noteType'],
        content,
      });
    }
    
    return result;
  }

  /**
   * Normalize document totals
   */
  private normalizeDocumentTotals(
    totals: unknown,
    lineItems: ParsedLineItem[]
  ): ParsedDocumentTotals {
    const calculated = lineItems.reduce((sum, item) => sum + item.totalPrice, 0);
    
    if (!totals || typeof totals !== 'object') {
      return {
        subtotal: null,
        tax: null,
        grandTotal: calculated,
      };
    }
    
    const obj = totals as Record<string, unknown>;
    
    return {
      subtotal: this.normalizeNumber(obj.subtotal),
      tax: this.normalizeNumber(obj.tax),
      grandTotal: this.normalizeNumber(obj.grandTotal) || calculated,
    };
  }
}

export class EstimateParseError extends Error {
  constructor(
    message: string,
    public readonly rawResponse?: string
  ) {
    super(message);
    this.name = 'EstimateParseError';
  }
}
```

### Extraction Prompt (prompts/extraction.ts)

```typescript
export const EXTRACTION_SYSTEM_PROMPT = `You are a construction estimate parsing expert. Your task is to extract structured data from construction estimates, bids, and proposals.

You must return ONLY valid JSON matching this exact structure. No markdown, no explanation, no preamble - just the JSON object.

REQUIRED OUTPUT SCHEMA:
{
  "contractor": {
    "name": "string or null - company name",
    "address": "string or null - full address",
    "phone": "string or null - phone number",
    "email": "string or null - email address"
  },
  "client": {
    "name": "string or null - customer/client name",
    "projectName": "string or null - project name or address"
  },
  "lineItems": [
    {
      "description": "string - EXACT text from document, do not paraphrase",
      "sectionHeader": "string or null - section this item belongs to (e.g., 'Framing', 'Electrical')",
      "quantity": "number or null",
      "unit": "string or null (e.g., SF, LF, EA, HR, LS)",
      "unitPrice": "number or null",
      "totalPrice": "number - REQUIRED, the extended/total amount"
    }
  ],
  "notes": [
    {
      "noteType": "note|exclusion|disclaimer|term",
      "content": "string - the note text"
    }
  ],
  "documentTotals": {
    "subtotal": "number or null",
    "tax": "number or null", 
    "grandTotal": "number - total estimate amount"
  }
}

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
   - If unit price × quantity = total is shown, extract all three

3. SECTIONS - Identify section headers:
   - Common sections: Demolition, Framing, Electrical, Plumbing, HVAC, etc.
   - Assign items to the most recent section header above them
   - If no clear section, leave sectionHeader as null

4. NOTES & EXCLUSIONS - Capture these separately:
   - "Exclusions:" sections → noteType: "exclusion"
   - "Notes:" or "Note:" → noteType: "note"  
   - "Terms:" or "Terms and Conditions" → noteType: "term"
   - General disclaimers → noteType: "disclaimer"

5. TOTALS - Find the final amounts:
   - Look for "Subtotal", "Tax", "Total", "Grand Total"
   - grandTotal should be the final amount due

6. CONTRACTOR INFO - Usually at top of document:
   - Company name, logo text
   - Address, phone, email
   - License number can be included in address field

7. CLIENT INFO - Usually below contractor:
   - Customer/client name
   - Project name or job site address

COMMON UNITS:
- SF = Square Feet
- LF = Linear Feet
- EA = Each
- HR = Hour
- LS = Lump Sum
- SY = Square Yard
- CY = Cubic Yard

Remember: Return ONLY the JSON object. No other text.`;

/**
 * Alternative prompt for simple estimates (fewer tokens)
 */
export const SIMPLE_EXTRACTION_PROMPT = `Extract data from this construction estimate as JSON:

{
  "contractor": {"name": null, "address": null, "phone": null, "email": null},
  "client": {"name": null, "projectName": null},
  "lineItems": [{"description": "text", "sectionHeader": null, "quantity": null, "unit": null, "unitPrice": null, "totalPrice": 0}],
  "notes": [{"noteType": "note", "content": "text"}],
  "documentTotals": {"subtotal": null, "tax": null, "grandTotal": 0}
}

Rules:
- Extract ALL line items with prices
- Keep EXACT descriptions
- Numbers only (no $ or commas)
- Return ONLY JSON`;
```

## Handling Edge Cases

### Retry Logic

```typescript
export class EstimateParserWithRetry extends EstimateParser {
  private readonly maxRetries = 2;
  
  async parse(rawText: string, filename: string): Promise<ParsedEstimate> {
    let lastError: Error | null = null;
    
    for (let attempt = 0; attempt <= this.maxRetries; attempt++) {
      try {
        return await super.parse(rawText, filename);
      } catch (error) {
        lastError = error instanceof Error ? error : new Error(String(error));
        
        // Don't retry on certain errors
        if (error instanceof EstimateParseError && error.rawResponse) {
          // If we got a response but it wasn't valid JSON,
          // try with simpler prompt
          if (attempt === 0) {
            console.log('Retrying with simplified prompt...');
            // Could implement simpler prompt here
          }
        }
        
        // Brief delay before retry
        if (attempt < this.maxRetries) {
          await new Promise(r => setTimeout(r, 500 * (attempt + 1)));
        }
      }
    }
    
    throw lastError || new Error('Parse failed after retries');
  }
}
```

### Fallback Extraction

```typescript
/**
 * Attempt basic extraction without LLM as fallback
 */
export function extractBasicLineItems(text: string): ParsedLineItem[] {
  const items: ParsedLineItem[] = [];
  const lines = text.split('\n');
  
  // Pattern: description followed by amount
  const amountPattern = /\$?([\d,]+\.?\d*)\s*$/;
  
  for (const line of lines) {
    const match = line.match(amountPattern);
    if (match) {
      const amount = parseFloat(match[1].replace(/,/g, ''));
      if (amount > 0 && amount < 10000000) {  // Sanity check
        const description = line.slice(0, match.index).trim();
        if (description.length > 3) {  // Reasonable description
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
  
  return items;
}
```

## Usage Example

```typescript
// In session processing

import { EstimateParser, EstimateParseError } from '../services/estimate-parser';

async function parseEstimate(
  env: Env,
  rawText: string,
  filename: string
): Promise<ParsedEstimate> {
  const parser = new EstimateParser(env.AI);
  
  try {
    return await parser.parse(rawText, filename);
  } catch (error) {
    if (error instanceof EstimateParseError) {
      console.error('Parse error:', error.message);
      console.error('Raw response:', error.rawResponse?.slice(0, 500));
      
      // Try fallback extraction
      const basicItems = extractBasicLineItems(rawText);
      if (basicItems.length > 0) {
        console.log(`Fallback extracted ${basicItems.length} items`);
        return {
          contractor: { name: null, address: null, phone: null, email: null },
          client: null,
          lineItems: basicItems,
          notes: [],
          documentTotals: {
            subtotal: null,
            tax: null,
            grandTotal: basicItems.reduce((s, i) => s + i.totalPrice, 0),
          },
        };
      }
    }
    
    throw error;
  }
}
```

## Todo List

### Core Parser

- [ ] Create estimate-parser.ts file
- [ ] Implement EstimateParser class
- [ ] Implement parse method
- [ ] Implement truncateText for long documents
- [ ] Implement buildPrompt method
- [ ] Implement extractResponseText method
- [ ] Implement parseResponse method
- [ ] Implement extractJSON (handle markdown blocks)
- [ ] Create EstimateParseError class

### Validation & Normalization

- [ ] Implement validateAndNormalize method
- [ ] Implement normalizeString helper
- [ ] Implement normalizeNumber helper
- [ ] Implement normalizeLineItems method
- [ ] Implement normalizeNotes method
- [ ] Implement normalizeDocumentTotals method
- [ ] Handle calculated totals when not provided

### Prompts

- [ ] Create prompts/extraction.ts file
- [ ] Write EXTRACTION_SYSTEM_PROMPT
- [ ] Write SIMPLE_EXTRACTION_PROMPT (fallback)
- [ ] Test prompt with various estimate formats
- [ ] Refine prompt based on results

### Error Handling

- [ ] Implement retry logic
- [ ] Implement fallback extraction (extractBasicLineItems)
- [ ] Handle malformed JSON responses
- [ ] Handle empty responses
- [ ] Handle timeout errors
- [ ] Log errors appropriately

### Testing

- [ ] Test with simple one-page estimate
- [ ] Test with multi-section estimate
- [ ] Test with table-formatted estimate
- [ ] Test with poorly structured estimate
- [ ] Test line item extraction accuracy
- [ ] Test number parsing (various formats)
- [ ] Test note/exclusion extraction
- [ ] Verify performance (response time)

## Verification Checklist

- [ ] Parser extracts contractor info correctly
- [ ] Parser extracts client info correctly
- [ ] All line items with prices are extracted
- [ ] Line item descriptions are preserved exactly
- [ ] Section headers are assigned correctly
- [ ] Numbers are parsed correctly (no $ or commas)
- [ ] Notes and exclusions are categorized
- [ ] Document totals are extracted or calculated
- [ ] Fallback extraction works when LLM fails
- [ ] Error messages are helpful

## Notes

### Workers AI Limitations

- Model: `@cf/meta/llama-3.1-8b-instruct`
- Max input: ~16K tokens (practical limit ~12K)
- Max output: 4K tokens
- Response time: 2-10 seconds typically

### Prompt Engineering Tips

1. **Be explicit about JSON format** - LLMs often add markdown
2. **Low temperature (0.1)** - More consistent structured output
3. **Few-shot examples** - Can improve accuracy but uses tokens
4. **Negative instructions** - "Do NOT paraphrase" helps

### Common Parsing Issues

| Issue | Solution |
|-------|----------|
| Missing line items | Check for alternative table formats |
| Wrong numbers | Verify currency format parsing |
| Merged items | Look for multi-line descriptions |
| Missing sections | May not have section headers |
| Truncated response | Reduce input size |

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
Upper cabinets 8 LF @ $125        $1,000.00

Subtotal:                         $3,600.00
Tax:                                $288.00
Total:                            $3,888.00

Exclusions:
- Permits not included
- Electrical work by others
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
    {"description": "Base cabinets", "sectionHeader": "CABINETS", "quantity": 12, "unit": "LF", "unitPrice": 150, "totalPrice": 1800.00},
    {"description": "Upper cabinets", "sectionHeader": "CABINETS", "quantity": 8, "unit": "LF", "unitPrice": 125, "totalPrice": 1000.00}
  ],
  "notes": [
    {"noteType": "exclusion", "content": "Permits not included"},
    {"noteType": "exclusion", "content": "Electrical work by others"}
  ],
  "documentTotals": {
    "subtotal": 3600.00,
    "tax": 288.00,
    "grandTotal": 3888.00
  }
}
```

## Time Estimate

| Task | Estimate |
|------|----------|
| Core parser | 2 hours |
| Validation/normalization | 1.5 hours |
| Prompts | 1 hour |
| Error handling | 1 hour |
| Testing | 1.5 hours |
| **Total** | **~7 hours** |
