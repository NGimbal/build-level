# Module 06: Classification

## Overview

LLM-based classification service that assigns cost codes to extracted line items. Uses Workers AI to match line item descriptions to the standardized cost code format.

## Goals

- Classify line items to appropriate cost codes
- Provide confidence scores for classifications
- Suggest alternate classifications
- Support batch classification for efficiency
- Handle ambiguous items gracefully

## File Structure

```
packages/backend/src/services/
├── classifier.ts           # Main classification service
└── prompts/
    └── classification.ts   # Classification prompt templates
```

## Core Implementation

### Classifier Service (classifier.ts)

```typescript
import type { Ai } from '@cloudflare/workers-types';
import type { CostCode } from '@estimate-compare/shared';

export interface ClassificationInput {
  id: string;
  description: string;
  sectionHeader: string | null;
  quantity?: number | null;
  unit?: string | null;
}

export interface ClassificationResult {
  lineItemId: string;
  code: string;
  confidence: number;
  alternates: Array<{ code: string; confidence: number }>;
}

export class Classifier {
  private readonly model = '@cf/meta/llama-3.1-8b-instruct';
  private readonly batchSize = 15;  // Items per LLM call
  
  constructor(private ai: Ai) {}

  /**
   * Classify multiple line items to cost codes
   */
  async classifyLineItems(
    items: ClassificationInput[],
    costCodes: CostCode[]
  ): Promise<Map<string, ClassificationResult>> {
    const results = new Map<string, ClassificationResult>();
    
    if (items.length === 0) {
      return results;
    }
    
    // Build code reference once
    const codeReference = this.buildCodeReference(costCodes);
    
    // Process in batches
    for (let i = 0; i < items.length; i += this.batchSize) {
      const batch = items.slice(i, i + this.batchSize);
      
      try {
        const batchResults = await this.classifyBatch(batch, codeReference, costCodes);
        
        for (const result of batchResults) {
          results.set(result.lineItemId, result);
        }
      } catch (error) {
        console.error(`Batch classification failed for items ${i}-${i + batch.length}:`, error);
        
        // Fall back to default classification for failed batch
        for (const item of batch) {
          results.set(item.id, this.createDefaultClassification(item, costCodes));
        }
      }
    }
    
    return results;
  }

  /**
   * Classify a single line item
   */
  async classifySingle(
    item: ClassificationInput,
    costCodes: CostCode[]
  ): Promise<ClassificationResult> {
    const results = await this.classifyLineItems([item], costCodes);
    return results.get(item.id) || this.createDefaultClassification(item, costCodes);
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
        lines.push(`  ${child.code}: ${child.label} (${l1.label})`);
      }
    }
    
    return lines.join('\n');
  }

  /**
   * Classify a batch of items
   */
  private async classifyBatch(
    items: ClassificationInput[],
    codeReference: string,
    costCodes: CostCode[]
  ): Promise<ClassificationResult[]> {
    const prompt = this.buildClassificationPrompt(items, codeReference);
    
    const response = await this.ai.run(this.model, {
      messages: [
        { role: 'system', content: CLASSIFICATION_SYSTEM_PROMPT },
        { role: 'user', content: prompt },
      ],
      max_tokens: 2000,
      temperature: 0.1,
    });

    const responseText = typeof response === 'string'
      ? response
      : (response as any).response;

    return this.parseClassifications(responseText, items, costCodes);
  }

  /**
   * Build prompt for batch classification
   */
  private buildClassificationPrompt(
    items: ClassificationInput[],
    codeReference: string
  ): string {
    const itemsList = items.map(item => {
      let line = `[${item.id}] ${item.description}`;
      
      if (item.sectionHeader) {
        line += ` [Section: ${item.sectionHeader}]`;
      }
      
      if (item.unit) {
        line += ` [Unit: ${item.unit}]`;
      }
      
      return line;
    }).join('\n');

    return `Classify these construction line items to cost codes.

AVAILABLE CODES:
${codeReference}

LINE ITEMS:
${itemsList}

Return ONLY a JSON array. No other text.`;
  }

  /**
   * Parse classification response
   */
  private parseClassifications(
    responseText: string,
    items: ClassificationInput[],
    costCodes: CostCode[]
  ): ClassificationResult[] {
    // Extract JSON array
    let jsonStr = responseText;
    
    const codeBlockMatch = responseText.match(/```(?:json)?\s*([\s\S]*?)```/);
    if (codeBlockMatch) {
      jsonStr = codeBlockMatch[1];
    }
    
    const arrayMatch = jsonStr.match(/\[[\s\S]*\]/);
    if (arrayMatch) {
      jsonStr = arrayMatch[0];
    }

    const validCodes = new Set(costCodes.map(c => c.code));
    
    try {
      const parsed = JSON.parse(jsonStr);
      
      if (!Array.isArray(parsed)) {
        throw new Error('Response is not an array');
      }
      
      // Map parsed results to items
      return items.map(item => {
        const match = parsed.find((p: any) => p.id === item.id);
        
        if (match && validCodes.has(match.code)) {
          // Validate alternates
          const alternates = Array.isArray(match.alternates)
            ? match.alternates
                .filter((a: any) => validCodes.has(a.code))
                .slice(0, 3)
                .map((a: any) => ({
                  code: a.code,
                  confidence: this.normalizeConfidence(a.confidence),
                }))
            : [];
          
          return {
            lineItemId: item.id,
            code: match.code,
            confidence: this.normalizeConfidence(match.confidence),
            alternates,
          };
        }
        
        // Item not in response or invalid code
        return this.createDefaultClassification(item, costCodes);
      });
      
    } catch (error) {
      console.error('Failed to parse classification response:', error);
      console.error('Response was:', responseText.slice(0, 500));
      
      // Return default classifications for all items
      return items.map(item => this.createDefaultClassification(item, costCodes));
    }
  }

  /**
   * Normalize confidence to 0-1 range
   */
  private normalizeConfidence(value: unknown): number {
    if (typeof value !== 'number' || isNaN(value)) {
      return 0.5;
    }
    
    // Handle if provided as percentage
    if (value > 1) {
      value = value / 100;
    }
    
    return Math.max(0, Math.min(1, value));
  }

  /**
   * Create default classification for failed items
   */
  private createDefaultClassification(
    item: ClassificationInput,
    costCodes: CostCode[]
  ): ClassificationResult {
    // Try to infer from section header
    const inferredCode = this.inferFromSection(item.sectionHeader, costCodes);
    
    return {
      lineItemId: item.id,
      code: inferredCode || '15',  // Default to General Conditions
      confidence: inferredCode ? 0.4 : 0.2,
      alternates: [],
    };
  }

  /**
   * Try to infer code from section header
   */
  private inferFromSection(
    section: string | null,
    costCodes: CostCode[]
  ): string | null {
    if (!section) return null;
    
    const sectionLower = section.toLowerCase();
    const level1Codes = costCodes.filter(c => c.level === 1);
    
    for (const code of level1Codes) {
      const labelLower = code.label.toLowerCase();
      const keywords = code.keywords || [];
      
      // Check label match
      if (sectionLower.includes(labelLower) || labelLower.includes(sectionLower)) {
        return code.code;
      }
      
      // Check keywords
      for (const keyword of keywords) {
        if (sectionLower.includes(keyword.toLowerCase())) {
          return code.code;
        }
      }
    }
    
    return null;
  }
}
```

### Classification Prompt (prompts/classification.ts)

```typescript
export const CLASSIFICATION_SYSTEM_PROMPT = `You are a construction cost classification expert. Your task is to assign cost codes to construction line items.

Return ONLY a JSON array with this structure (no other text):
[
  {
    "id": "item-id-from-input",
    "code": "best-matching-code",
    "confidence": 0.0-1.0,
    "alternates": [
      {"code": "second-choice", "confidence": 0.0-1.0}
    ]
  }
]

CLASSIFICATION RULES:

1. CODE SELECTION
   - Use the MOST SPECIFIC code that applies (level 2 over level 1)
   - Consider the section header as strong context
   - When in doubt, use the parent division code

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
   - Section header strongly suggests the division
   - Units like "HR" suggest labor
   - "Subcontract" or "Sub" in description → _S codes
   - Material names (lumber, pipe, wire) → _M codes
   - "Install" or "labor" → _L codes

5. ALTERNATES
   - Provide 1-2 alternate codes when classification is uncertain
   - Alternates should be plausible alternatives

COMMON MAPPINGS:

Division 00 (Demolition): tear out, remove, demo, abatement, gut, strip
Division 01 (Sitework): grading, excavation, landscaping, dirt, paving
Division 02 (Foundation): footing, slab, concrete foundation, stem wall
Division 03 (Framing): studs, joists, rafters, trusses, sheathing, deck framing
Division 04 (Roofing): shingles, roofing, flashing, gutters, soffit, fascia
Division 05 (Windows & Doors): windows, doors, entry, sliding door, hardware
Division 06 (Siding & Trim): siding, exterior trim, stucco, brick veneer
Division 07 (Electrical): wiring, panel, outlets, switches, lighting, fixtures
Division 08 (HVAC): furnace, AC, ductwork, ventilation, heat pump
Division 09 (Plumbing): pipes, fixtures, water heater, drains, faucets
Division 10 (Insulation): insulation, batt, blown-in, spray foam
Division 11 (Carpentry): finish carpentry, trim, molding, stairs, railings
Division 12 (Casework): cabinets, countertops, vanities
Division 13 (Drywall & Finishes): drywall, paint, tile, flooring
Division 14 (Specialties): appliances, fireplace, mirrors, accessories
Division 15 (General Conditions): supervision, permits, cleanup, temporary

Remember: Return ONLY the JSON array.`;
```

## Advanced Classification

### Confidence-Based Reclassification

```typescript
/**
 * Get items that may need human review
 */
export function getLowConfidenceItems(
  results: Map<string, ClassificationResult>,
  threshold: number = 0.6
): ClassificationResult[] {
  return Array.from(results.values())
    .filter(r => r.confidence < threshold)
    .sort((a, b) => a.confidence - b.confidence);
}

/**
 * Calculate classification quality metrics
 */
export function calculateClassificationMetrics(
  results: Map<string, ClassificationResult>
): {
  totalItems: number;
  highConfidence: number;
  mediumConfidence: number;
  lowConfidence: number;
  averageConfidence: number;
} {
  const values = Array.from(results.values());
  const total = values.length;
  
  if (total === 0) {
    return {
      totalItems: 0,
      highConfidence: 0,
      mediumConfidence: 0,
      lowConfidence: 0,
      averageConfidence: 0,
    };
  }
  
  const high = values.filter(v => v.confidence >= 0.8).length;
  const medium = values.filter(v => v.confidence >= 0.5 && v.confidence < 0.8).length;
  const low = values.filter(v => v.confidence < 0.5).length;
  const avgConf = values.reduce((sum, v) => sum + v.confidence, 0) / total;
  
  return {
    totalItems: total,
    highConfidence: high,
    mediumConfidence: medium,
    lowConfidence: low,
    averageConfidence: avgConf,
  };
}
```

### Batch Reclassification

```typescript
/**
 * Reclassify items that were assigned to a specific code
 */
export async function reclassifyItemsWithCode(
  classifier: Classifier,
  items: ClassificationInput[],
  oldCode: string,
  newCode: string,
  costCodes: CostCode[]
): Promise<Map<string, ClassificationResult>> {
  const itemsToReclassify = items.filter(item => {
    // This would need the current classifications
    return true;  // Placeholder
  });
  
  // Re-run classification for affected items
  return classifier.classifyLineItems(itemsToReclassify, costCodes);
}
```

## Usage in Processing Pipeline

```typescript
// In session processing

import { Classifier } from '../services/classifier';
import { getCostCodesByFormat } from '../db/queries';

async function classifyEstimateItems(
  env: Env,
  estimateId: string,
  lineItems: Array<{ id: string; description: string; sectionHeader: string | null }>
): Promise<Map<string, ClassificationResult>> {
  // Get session to find format
  const estimate = await getEstimate(env.DB, estimateId);
  const session = await getSession(env.DB, estimate.sessionId);
  
  // Load cost codes
  const costCodes = await getCostCodesByFormat(env.DB, session.formatId);
  
  // Classify
  const classifier = new Classifier(env.AI);
  const classifications = await classifier.classifyLineItems(
    lineItems.map(item => ({
      id: item.id,
      description: item.description,
      sectionHeader: item.sectionHeader,
    })),
    costCodes
  );
  
  // Log metrics
  const metrics = calculateClassificationMetrics(classifications);
  console.log('Classification metrics:', metrics);
  
  return classifications;
}
```

## Todo List

### Core Classifier

- [ ] Create classifier.ts file
- [ ] Implement Classifier class
- [ ] Implement classifyLineItems method (batch)
- [ ] Implement classifySingle method
- [ ] Implement buildCodeReference method
- [ ] Implement classifyBatch method
- [ ] Implement buildClassificationPrompt method

### Response Parsing

- [ ] Implement parseClassifications method
- [ ] Handle JSON extraction from markdown
- [ ] Validate codes against valid code list
- [ ] Normalize confidence values
- [ ] Handle missing/invalid items in response

### Fallback Logic

- [ ] Implement createDefaultClassification method
- [ ] Implement inferFromSection method
- [ ] Test fallback with various section headers
- [ ] Handle batch failures gracefully

### Prompts

- [ ] Create prompts/classification.ts file
- [ ] Write CLASSIFICATION_SYSTEM_PROMPT
- [ ] Include all division mappings
- [ ] Test prompt with various line items
- [ ] Refine prompt based on results

### Utilities

- [ ] Implement getLowConfidenceItems function
- [ ] Implement calculateClassificationMetrics function
- [ ] Add logging for classification quality

### Testing

- [ ] Test with clear division assignments
- [ ] Test with ambiguous items
- [ ] Test cost type classification (L/M/S)
- [ ] Test batch processing
- [ ] Test single item classification
- [ ] Test fallback behavior
- [ ] Verify confidence scores are reasonable
- [ ] Test with real estimate line items

## Verification Checklist

- [ ] Classifications match expected codes
- [ ] Confidence scores are reasonable
- [ ] Alternates are provided for uncertain items
- [ ] Section headers improve classification
- [ ] Batch processing handles errors gracefully
- [ ] All items get a classification (no gaps)
- [ ] Invalid codes are rejected
- [ ] Performance is acceptable (<5s per batch)

## Notes

### Classification Accuracy

Expected accuracy by category:

| Category | Expected Accuracy |
|----------|-------------------|
| Division (Level 1) | 85-95% |
| Cost Type (Level 2) | 70-85% |
| Overall | 75-85% |

### Common Misclassifications

| Item Type | Likely Confusion |
|-----------|------------------|
| "Install X" | Labor vs Material |
| Subcontractor work | Division vs General Conditions |
| Cleanup/Debris | Demolition vs General Conditions |
| Misc materials | Wrong division |

### Improving Classification

1. **Section Headers**: Strong signal - ensure parsed correctly
2. **Keywords**: Add more keywords to cost code definitions
3. **User Feedback**: Track manual overrides to improve
4. **Few-shot Examples**: Could add to prompt if needed

### Sample Input → Output

**Input:**
```
[item-1] Install kitchen base cabinets [Section: CASEWORK]
[item-2] 2x4 studs [Section: FRAMING] [Unit: BF]
[item-3] Electrical subcontract - rough and finish
[item-4] Labor for drywall hanging
[item-5] Cleanup and debris removal
```

**Output:**
```json
[
  {"id": "item-1", "code": "12_L", "confidence": 0.85, "alternates": [{"code": "12_S", "confidence": 0.6}]},
  {"id": "item-2", "code": "03_M", "confidence": 0.95, "alternates": []},
  {"id": "item-3", "code": "07_S", "confidence": 0.90, "alternates": []},
  {"id": "item-4", "code": "13_L", "confidence": 0.88, "alternates": [{"code": "13_S", "confidence": 0.5}]},
  {"id": "item-5", "code": "00_L", "confidence": 0.65, "alternates": [{"code": "15_L", "confidence": 0.55}]}
]
```

## Time Estimate

| Task | Estimate |
|------|----------|
| Core classifier | 2 hours |
| Response parsing | 1.5 hours |
| Fallback logic | 1 hour |
| Prompts | 45 min |
| Utilities | 30 min |
| Testing | 1.5 hours |
| **Total** | **~7.25 hours** |
