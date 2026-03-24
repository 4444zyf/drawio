# Hybrid Grid-Graph Algorithm for LLM Element Position Understanding

**Date**: 2026-03-24
**Status**: Draft
**Authors**: Claude + User Collaboration

## Executive Summary

This document describes an algorithm designed to help Large Language Models (LLMs) and AI Agents better understand, describe, generate, and validate element positions in draw.io diagrams. The algorithm combines a **Semantic Grid** for coarse positioning with a **Spatial Relationship Graph** for fine-grained spatial reasoning.

## Problem Statement

### Challenges Addressed

When LLMs interact with draw.io diagrams, they face several challenges:

1. **Spatial Reasoning Issues**: LLMs struggle to reason about spatial relationships (above, below, left, right) from raw coordinate data
2. **Coordinate Mismatch**: LLM output often doesn't match the diagram's coordinate system or scale
3. **Invalid Diagram Generation**: LLMs may generate invalid XML or incorrect positions when creating diagrams

### Use Cases

- **Understanding**: LLM answers questions about element positions and relationships
- **Describing**: LLM generates natural language descriptions of diagram layout
- **Generating**: LLM creates or modifies diagrams by specifying element positions
- **Tracking**: Maintain position context across long conversations

## Algorithm Overview

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Input Layer                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │ .drawio XML │  │ mxGraphModel│  │ Live Editor │          │
│  │    File     │  │   Object    │  │   State     │          │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘          │
│         └────────────────┼────────────────┘                  │
│                          ▼                                   │
├─────────────────────────────────────────────────────────────┤
│                 Processing Pipeline                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   Parser    │─▶│  Grid       │─▶│  Graph      │          │
│  │  Module     │  │  Indexer    │  │  Builder    │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
│                          │                                   │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────┐            │
│  │           Spatial Context Generator          │            │
│  └─────────────────────────────────────────────┘            │
├─────────────────────────────────────────────────────────────┤
│                    Output Layer                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │ LLM Context │  │  Position   │  │   Diagram   │          │
│  │   Enriched  │  │ Description │  │  Generation │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

### Core Components

1. **Parser Module**: Converts XML/mxGraphModel to normalized element representation
2. **Grid Indexer**: Maps elements to semantic grid cells
3. **Graph Builder**: Computes pairwise spatial relationships
4. **Spatial Context Generator**: Produces LLM-friendly context strings
5. **Position Validator**: Validates and corrects generated positions

## Data Model

### draw.io Element Representation

draw.io uses the following structure for elements:

```xml
<mxCell id="2" value="Button" vertex="1" style="rounded=1;">
  <mxGeometry x="100" y="50" width="80" height="30" as="geometry"/>
</mxCell>
```

Key attributes:
- `id`: Unique identifier
- `value`: Element label/text
- `vertex/edge`: Element type
- `source/target`: For edges, references to connected elements
- `x, y, width, height`: Position and dimensions in pixels

### Normalized Element Representation

```typescript
interface DiagramElement {
  id: string;                    // mxCell.id
  type: 'vertex' | 'edge' | 'group';
  label: string;                 // value attribute

  // Position data
  geometry: {
    x: number;                   // absolute x coordinate
    y: number;                   // absolute y coordinate
    width: number;
    height: number;
    normalizedX: number;         // 0-1 range
    normalizedY: number;         // 0-1 range
  };

  // Grid mapping
  gridPosition: {
    row: number;                 // 0-based row index
    col: number;                 // 0-based column index
    cellName: string;            // e.g., "B3", "C5"
    semanticZone: string;        // e.g., "top-left", "center"
  };

  // Relationships (for edges)
  connections: {
    source?: string;             // source cell id
    target?: string;             // target cell id
    points?: Point[];            // control points
  };

  // Style info for semantic understanding
  style: {
    shape: string;               // e.g., "rectangle", "ellipse"
    fillColor?: string;
    strokeColor?: string;
  };
}
```

### Semantic Grid Configuration

```typescript
interface GridConfig {
  rows: number;                  // default: 10
  cols: number;                  // default: 10

  // Zone definitions (as percentage of grid)
  zones: {
    name: string;                // e.g., "header", "sidebar"
    rowRange: [number, number];  // [start, end]
    colRange: [number, number];
  }[];

  // Cell naming scheme
  namingScheme: 'excel' | 'chess' | 'semantic';
  // excel: A1, B2, C3...
  // chess: a8, b7... (board notation)
  // semantic: top-left, center, bottom-right...
}
```

### Spatial Relationship Graph

```typescript
interface SpatialGraph {
  nodes: Map<string, DiagramElement>;
  edges: SpatialRelation[];
  adjacencyList: Map<string, Set<string>>;
}

interface SpatialRelation {
  sourceId: string;
  targetId: string;
  relationType:
    | 'above'           // source.y < target.y
    | 'below'           // source.y > target.y
    | 'left-of'         // source.x < target.x
    | 'right-of'        // source.x > target.x
    | 'contains'        // source bounds contain target
    | 'contained-by'    // inverse of contains
    | 'overlaps'        // bounds intersect
    | 'connected-to'    // edge relationship
    | 'aligned-h'       // horizontally aligned
    | 'aligned-v';      // vertically aligned

  distance: number;     // pixel distance between centers
  confidence: number;   // 0-1, for ambiguous cases
}
```

## Algorithm Details

### Phase 1: Parsing & Normalization

**Input**: mxGraphModel XML or Object
**Output**: List of DiagramElement

```
Algorithm:
1. Extract root element and traverse cell hierarchy
2. For each mxCell:
   a. Parse geometry (x, y, width, height)
   b. Extract style string and parse into key-value pairs
   c. Determine element type (vertex/edge/group)
   d. Extract label from value attribute
   e. For edges: extract source, target, control points
3. Calculate global bounds (minX, maxX, minY, maxY)
4. Normalize all coordinates to [0,1] range
5. Return sorted list (by y then x for reading order)
```

### Phase 2: Grid Indexing

**Input**: List of DiagramElement, GridConfig
**Output**: Elements with gridPosition populated

```
Algorithm:
1. Divide canvas into grid cells:
   cellWidth = canvasWidth / cols
   cellHeight = canvasHeight / rows

2. For each element:
   a. Calculate center point: (x + width/2, y + height/2)
   b. Map to grid cell:
      row = floor(centerY / cellHeight)
      col = floor(centerX / cellWidth)
   c. Clamp to valid range [0, rows-1] × [0, cols-1]
   d. Generate cell name (e.g., "B3" for col=1, row=2)
   e. Determine semantic zone from zone definitions

3. Build cell-to-element index for O(1) lookup
```

### Phase 3: Spatial Graph Construction

**Input**: List of DiagramElement
**Output**: SpatialGraph

```
Algorithm:
1. Initialize empty graph with all elements as nodes

2. For each pair of elements (ei, ej) where i < j:
   a. Compute bounding boxes
   b. Determine spatial relationship:
      - Check vertical alignment (|centerY diff| < threshold)
      - Check horizontal alignment (|centerX diff| < threshold)
      - Check above/below (centerY comparison)
      - Check left/right (centerX comparison)
      - Check contains (bounding box inclusion)
      - Check overlaps (bounding box intersection)
   c. For edges: add 'connected-to' relation

3. For efficiency, use spatial hashing:
   - Only compare elements within nearby grid cells
   - Reduces complexity from O(n²) to O(n × k) where k = avg neighbors

4. Build adjacency list for graph traversal
```

### Phase 4: Context Generation

**Input**: DiagramElement[], SpatialGraph, GridConfig
**Output**: LLMContextOutput

```
Algorithm:
1. Generate Grid Report (Markdown):
   # Element Positions (10×10 Grid)

   | Cell | Elements |
   |------|----------|
   | A1 | Header (x:50, y:20) |
   | B3 | Submit Button (x:150, y:120) |
   | C5 | Cancel Button (x:250, y:200) |

2. Generate Relationship Report (Natural Language):
   Spatial Relationships:
   - "Submit Button" is below "Header" (distance: 100px)
   - "Submit Button" is left of "Cancel Button" (distance: 50px)
   - Both buttons are horizontally aligned
   - "Submit Button" is in the top-left quadrant

3. Generate Validation Rules:
   - Element must not overlap with [list of existing elements]
   - Element must be within bounds [minX, maxX, minY, maxY]
   - Element must maintain grid alignment (optional)

4. Package structured data for programmatic use
```

### Phase 5: Position Validation (for generation)

**Input**: Proposed position, ValidationRule[]
**Output**: Valid position or error

```
Algorithm:
1. Check bounds: proposed position within canvas?
2. Check overlaps: does position overlap with other elements?
3. Check constraints: satisfies any user-defined constraints?
4. If invalid:
   a. Calculate nearest valid position
   b. Apply grid snapping if enabled
   c. Return corrected position with warnings
```

## API Design

### Core Functions

```typescript
// Main entry point - analyze diagram
function analyzeDiagram(
  input: string | mxGraphModel,
  config?: Partial<GridConfig>
): LLMContextOutput;

// Quick position lookup
function getElementsInCell(gridCell: string): DiagramElement[];

// Spatial queries
function getElementsAbove(elementId: string): DiagramElement[];
function getElementsBelow(elementId: string): DiagramElement[];
function getElementsLeftOf(elementId: string): DiagramElement[];
function getElementsRightOf(elementId: string): DiagramElement[];
function getNearestElement(elementId: string, direction: Direction): DiagramElement | null;

// Relationship queries
function getRelationship(elementA: string, elementB: string): SpatialRelation[];
function getPath(elementA: string, elementB: string): DiagramElement[];

// Generation support
function validatePosition(position: Position, rules?: ValidationRule[]): ValidationResult;
function suggestPosition(element: Partial<DiagramElement>, constraints?: Constraints): Position;
function snapToGrid(position: Position): Position;
```

### LLM Integration APIs

```typescript
// Generate context string for LLM prompt
function generateLLMContext(
  diagram: LLMContextOutput,
  format: 'markdown' | 'json' | 'text'
): string;

// Parse LLM response and extract position information
function parseLLMPositionResponse(response: string): PositionUpdate[];

// Apply position updates to diagram
function applyPositionUpdates(model: mxGraphModel, updates: PositionUpdate[]): mxGraphModel;
```

### Configuration Options

```typescript
interface AlgorithmConfig {
  grid: {
    rows: number;              // default: 10
    cols: number;              // default: 10
    adaptive: boolean;         // auto-adjust based on element density
    namingScheme: 'excel' | 'chess' | 'semantic';
  };

  relationship: {
    alignmentThreshold: number;  // pixels, default: 10
    considerEdges: boolean;       // include edges in graph
    maxRelations: number;         // limit relations per element
  };

  output: {
    includeCoordinates: boolean;
    includeStyles: boolean;
    language: 'en' | 'zh' | 'ja';
    verboseLevel: 'minimal' | 'normal' | 'detailed';
  };

  validation: {
    allowOverlap: boolean;
    snapToGrid: boolean;
    padding: number;            // minimum spacing between elements
  };
}
```

## LLM Prompt Templates

### Template 1: Position Understanding

```
You are analyzing a draw.io diagram. Here is the spatial context:

{gridReport}

{relationshipReport}

Question: {userQuestion}

Instructions: Use the grid positions and relationships to answer accurately.
```

### Template 2: Position Generation

```
You are generating a draw.io diagram. The current canvas is:

{gridReport}

Validation Rules:
{validationRules}

Task: {generationTask}

Instructions:
1. Specify positions using grid cells (e.g., "place in cell B3")
2. The algorithm will convert to exact coordinates
3. Validate against overlap rules
```

## Edge Cases

| Edge Case | Handling Strategy |
|-----------|-------------------|
| Empty diagram | Return minimal context with grid layout |
| Overlapping elements | Flag in validation, include in overlap relations |
| Grouped elements | Treat group as container, compute relative positions |
| Large diagrams (>100 elements) | Use spatial hashing, summarize by grid zone |
| Edges without geometry | Compute from source/target positions |
| Relative positioning | Convert to absolute using parent bounds |
| Rotated elements | Use bounding box after rotation |

## Performance Considerations

### Optimization Strategies

1. **Spatial Hashing**
   - Hash elements to grid cells for O(1) neighbor lookup
   - Only compute relations within neighboring cells

2. **Incremental Updates**
   - Cache computed positions
   - Re-compute only for changed elements in live editor

3. **Lazy Evaluation**
   - Compute relationship graph on demand
   - Pre-compute only grid positions (cheaper)

### Complexity Analysis

| Operation | Time Complexity | Space Complexity |
|-----------|-----------------|------------------|
| Parse | O(n) | O(n) |
| Grid Index | O(n) | O(n) |
| Graph Build (naive) | O(n²) | O(n²) |
| Graph Build (spatial hash) | O(n × k) | O(n × k) |
| Context Generation | O(n) | O(n) |

Where n = number of elements, k = average neighbors per cell

## Implementation Phases

| Phase | Scope | Deliverables |
|-------|-------|--------------|
| Phase 1 | Parser + Grid Indexer | Core parsing, normalization, grid mapping |
| Phase 2 | Graph Builder + Relationships | Spatial relationship computation |
| Phase 3 | Context Generator + LLM Templates | LLM integration layer |
| Phase 4 | Validator + Position Corrector | Generation support |
| Phase 5 | Live Editor Integration | Real-time updates, event handling |

## Future Enhancements

1. **Adaptive Grid**: Automatically adjust grid granularity based on element density
2. **Semantic Labeling**: Identify common UI patterns (forms, tables, navbars)
3. **Multi-page Support**: Handle diagrams with multiple pages/tabs
4. **Constraint Inference**: Learn layout constraints from existing diagrams
5. **Cross-diagram Learning**: Transfer position knowledge between similar diagrams

## References

- [draw.io Architecture](../CLAUDE.md)
- [mxGraph Documentation](https://jgraph.github.io/mxgraph/)
- [mxCell API](src/main/webapp/mxgraph/src/model/mxCell.js)
- [mxGeometry API](src/main/webapp/mxgraph/src/model/mxGeometry.js)