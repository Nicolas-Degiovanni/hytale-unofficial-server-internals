---
description: Architectural reference for AnchorDensity
---

# AnchorDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient Node

## Definition
```java
// Signature
public class AnchorDensity extends Density {
```

## Architecture & Concepts
The AnchorDensity class is a spatial transformation node within the procedural world generation's density graph. Its sole purpose is to translate, or shift, the coordinate system for all its descendant nodes. This is a fundamental component for creating large-scale, position-independent features.

It operates by reading a `densityAnchor` vector from the processing `Context`. This anchor represents a dynamic, temporary origin point for a specific feature being generated. The node then calculates a new sampling position relative to this anchor and passes a new, modified context to its child `input` node.

This mechanism allows the generator to define a complex shape (like a cave system, biome feature, or structure) once, and then place it anywhere in the world without rewriting its logic. The feature's internal geometry is always calculated relative to its own anchor, not the world's absolute origin (0,0,0).

The `isReversed` flag provides an inversion mechanism, allowing the transformation to be either subtracted from or added to the current position. This is useful for nested transformations or for mapping coordinates between different spaces.

## Lifecycle & Ownership
- **Creation:** AnchorDensity nodes are not instantiated directly in game logic. They are created by a higher-level graph builder, typically when the world generator parses its configuration presets from data files at server startup.
- **Scope:** The object's lifetime is bound to the density graph it is a part of. It persists as long as the world generator is active.
- **Destruction:** The object is marked for garbage collection when the world generator is reconfigured or the server shuts down, causing the root of the density graph to be dereferenced. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** This class is stateful. It holds a reference to its child `input` node, which is mutable via the `setInputs` method. The `isReversed` flag is immutable and fixed at construction. The class does not cache any computed values.

- **Thread Safety:** **Conditionally Safe.** The `process` method is re-entrant and safe for concurrent execution by multiple world generation worker threads. This safety is guaranteed because it creates a new, isolated `Density.Context` for its child call, preventing state contamination between threads.

    **WARNING:** The class itself is not fully thread-safe. The graph structure is mutable via `setInputs`. Modifying the graph by calling `setInputs` while worker threads are actively calling `process` will result in catastrophic race conditions and undefined behavior. The entire density graph must be treated as immutable after its initial construction.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Context context) | double | O(C) | Calculates the density by transforming the context's position relative to the context's anchor, then delegating to the child node. Complexity is dependent on the child graph (C). If no anchor is present, this node becomes a pass-through. |
| setInputs(Density[] inputs) | void | O(1) | Sets the child node for this transformation. Expects an array with exactly one element. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by most developers. It is declared and configured within world generation data files, which are then parsed by the engine to construct the density graph. The engine is solely responsible for invoking the `process` method.

```java
// Engine-level invocation during world generation
// context is pre-populated with an anchor by a feature generator

double densityValue = someAnchorDensityNode.process(context);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new AnchorDensity()`. These nodes must be constructed by the framework responsible for building the complete density graph to ensure proper integration.
- **State Mutation During Processing:** Never call `setInputs` on any node in the density graph after the world generator has started its work. The graph must be considered read-only during the generation phase.
- **Missing Anchor:** Relying on this node for a transformation without ensuring the upstream logic populates `context.densityAnchor` will cause the node to silently fail and act as a simple pass-through. This can lead to subtle bugs where features generate with incorrect shapes or at the wrong locations.

## Data Pipeline
The primary function of this class is to manipulate the coordinate data within the `Density.Context` before it is consumed by downstream nodes.

> Flow:
> Upstream Node -> `process(Context)` -> **AnchorDensity** reads `context.densityAnchor` -> Creates `childContext` with `position` = `position` +/- `anchor` -> `input.process(childContext)` -> Returns density value.

