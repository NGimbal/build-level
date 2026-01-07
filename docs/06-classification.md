# Module 06: Classification

## Overview

LLM-based classification service that assigns cost codes to extracted line items. Uses [BAML](https://github.com/BoundaryML/baml) for type-safe batch classification with Workers AI (Llama 3.1).

## Goals

- Classify line items to appropriate cost codes
- Provide confidence scores for classifications
- Suggest alternate classifications
- Support batch classification for efficiency
- Handle ambiguous items gracefully

## File Structure

```
build-level/
├── baml_src/
│   ├── clients.baml             # Shared with extraction (Module 05)
│   ├── types.baml               # Shared types + classification types
│   ├── classification.baml      # Classification functions
│   └── tests/
│       └── classification_tests.baml
├── baml_client/                 # Generated TypeScript
└── packages/backend/src/services/
    └── classifier.ts            # Classifier service using BAML
```

## Type Definitions (baml_src/types.baml)

Add to existing types.baml from Module 05:

```baml
// Classification input - line item to classify
class ClassificationInput {
  id string @description("Line item ID")
  description string @description("Line item description text")
  sectionHeader string? @description("Section header if available")
  unit string? @description("Unit of measure if available")
}

// Classification result
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

## Classification Function (baml_src/classification.baml)

```baml
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

    Use these confidence levels:
    - 0.90-1.00: Exact match
    - 0.70-0.89: Good match
    - 0.50-0.69: Reasonable match
    - Below 0.50: Uncertain

    {{ ctx.output_format }}
  "#
}
```

## Core Implementation

### Classifier Service (classifier.ts)

```typescript
import { b } from '../../baml_client';
import type { ClassificationResult, ClassificationInput } from '../../baml_client/types';
import type { CostCode } from '@estimate-compare/shared';

export class Classifier {
  private readonly batchSize = 15;  // Items per LLM call

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
            // Invalid code returned - use fallback
            const item = batch.find(item => item.id === result.id);
            if (item) {
              results.set(result.id, this.createDefaultClassification(item, costCodes));
            }
          }
        }

        // Handle any missing items in response
        for (const item of batch) {
          if (!results.has(item.id)) {
            results.set(item.id, this.createDefaultClassification(item, costCodes));
          }
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
   * Classify a single line item (for reclassification UI)
   */
  async classifySingle(
    description: string,
    sectionHeader: string | null,
    costCodes: CostCode[]
  ): Promise<ClassificationResult> {
    const codeReference = this.buildCodeReference(costCodes);

    try {
      const result = await b.ClassifySingleItem(
        description,
        sectionHeader,
        codeReference
      );

      // Validate code exists
      const validCodes = new Set(costCodes.map(c => c.code));
      if (validCodes.has(result.code)) {
        return result;
      }

      // Invalid code - return with fallback
      return this.createDefaultClassification(
        { id: 'single', description, sectionHeader },
        costCodes
      );
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
   * Create default classification for failed items
   */
  private createDefaultClassification(
    item: ClassificationInput,
    costCodes: CostCode[]
  ): ClassificationResult {
    // Try to infer from section header
    const inferredCode = this.inferFromSection(item.sectionHeader, costCodes);

    return {
      id: item.id,
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

## Quality Metrics

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

## Test Cases (baml_src/tests/classification_tests.baml)

```baml
test MixedLineItems {
  functions [ClassifyLineItems]
  args {
    items [
      {id: "item-1", description: "Install kitchen base cabinets", sectionHeader: "CASEWORK", unit: null},
      {id: "item-2", description: "2x4 studs", sectionHeader: "FRAMING", unit: "BF"},
      {id: "item-3", description: "Electrical subcontract - rough and finish", sectionHeader: null, unit: null},
      {id: "item-4", description: "Labor for drywall hanging", sectionHeader: "DRYWALL", unit: "HR"},
      {id: "item-5", description: "Cleanup and debris removal", sectionHeader: null, unit: null}
    ]
    costCodeReference #"
      00: Demolition
        00_L: Labor
        00_M: Material
        00_S: Subcontracts
      03: Framing
        03_L: Labor
        03_M: Material
        03_S: Subcontracts
      07: Electrical
        07_L: Labor
        07_M: Material
        07_S: Subcontracts
      12: Casework
        12_L: Labor
        12_M: Material
        12_S: Subcontracts
      13: Drywall & Finishes
        13_L: Labor
        13_M: Material
        13_S: Subcontracts
      15: General Conditions
        15_L: Labor
        15_M: Material
        15_S: Subcontracts
    "#
  }
}

test SingleItemClassification {
  functions [ClassifySingleItem]
  args {
    description "Install recessed lighting fixtures"
    sectionHeader "ELECTRICAL"
    costCodeReference #"
      07: Electrical
        07_L: Labor
        07_M: Material
        07_S: Subcontracts
    "#
  }
}
```

## Usage in Processing Pipeline

```typescript
import { Classifier, calculateClassificationMetrics } from '../services/classifier';
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
  const classifier = new Classifier();
  const classifications = await classifier.classifyLineItems(
    lineItems.map(item => ({
      id: item.id,
      description: item.description,
      sectionHeader: item.sectionHeader,
      unit: null,
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

### BAML Functions

- [ ] Add classification types to `types.baml`
- [ ] Create `classification.baml` with ClassifyLineItems function
- [ ] Create ClassifySingleItem function for reclassification UI
- [ ] Run `baml generate` to update TypeScript client

### Classifier Service

- [ ] Create classifier.ts using BAML client
- [ ] Implement classifyLineItems with batch processing
- [ ] Implement classifySingle for reclassification
- [ ] Implement buildCodeReference method
- [ ] Implement createDefaultClassification fallback
- [ ] Implement inferFromSection helper

### Quality Utilities

- [ ] Implement getLowConfidenceItems function
- [ ] Implement calculateClassificationMetrics function
- [ ] Add logging for classification quality

### Testing

- [ ] Create test cases in `baml_src/tests/`
- [ ] Test with clear division assignments
- [ ] Test with ambiguous items
- [ ] Test cost type classification (L/M/S)
- [ ] Test batch processing
- [ ] Test single item classification
- [ ] Test fallback behavior
- [ ] Verify confidence scores are reasonable

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
3. **User Feedback**: Track manual overrides to improve prompts
4. **Few-shot Examples**: Can add to prompt if needed

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
| BAML functions | 1 hour |
| Classifier service | 1.5 hours |
| Quality utilities | 30 min |
| Testing | 1.5 hours |
| **Total** | **~4.5 hours** |
