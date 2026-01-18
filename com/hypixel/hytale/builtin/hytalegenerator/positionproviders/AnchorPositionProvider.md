---
description: Architectural reference for AnchorPositionProvider
---

# AnchorPositionProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.positionproviders
**Type:** Transient Component

## Definition
```java
// Signature
public class AnchorPositionProvider extends PositionProvider {
```

## Architecture & Concepts
The AnchorPositionProvider is a decorator within the procedural world generation framework. It does not generate positions itself; instead, it modifies the coordinate space for a wrapped, or "child", PositionProvider.

Its primary function is to enable relative positioning. It achieves this by taking an "anchor" point from the execution Context and using it to translate the generation bounds (the bounding box) before passing the request to the child provider. When the child provider yields a position, this class translates it back into the original coordinate space.

This pattern is fundamental for composing complex world generation rules, such as placing a cluster of flowers relative to a tree, or spawning a dungeon entrance relative to a mountain peak. The `isReversed` flag controls the direction of the translation, allowing for both additive and subtractive anchoring.

## Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor, typically during the configuration phase of a larger world generation system. It is designed to be composed with other PositionProvider instances in a chain or tree structure.
- **Scope:** The lifetime of an AnchorPositionProvider is tied to the parent generator or configuration object that holds a reference to it. It is a short-lived object, existing only for the duration of a specific generation task.
- **Destruction:** The object is managed by the Java garbage collector. There are no native resources or explicit cleanup methods. It is reclaimed once the root of the generation task is no longer referenced.

## Internal State & Concurrency
- **State:** The internal state, consisting of the child `positionProvider` and the `isReversed` flag, is immutable. Both fields are marked as final and are set only during construction. The class itself is stateless during execution; all necessary information is passed via the `Context` parameter in the `positionsIn` method.
- **Thread Safety:** This class is thread-safe and re-entrant. Each call to `positionsIn` operates on its own stack and creates new instances of `Vector3d` and `Context` for the child provider. Concurrency safety is therefore delegated to the wrapped `positionProvider`. The presence of a `workerId` in the `Context` object strongly implies that the entire PositionProvider system is designed for parallel execution.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| AnchorPositionProvider(provider, isReversed) | constructor | O(1) | Constructs a new provider, wrapping the supplied child provider. |
| positionsIn(context) | void | Delegates | Translates the context, invokes the child provider, and translates the results back. Complexity is determined by the wrapped provider. |

## Integration Patterns

### Standard Usage
The AnchorPositionProvider is used to wrap another provider to offset its generation area and results. This is a core composition pattern in the generator system.

```java
// Assume 'gridProvider' generates points on a grid
PositionProvider gridProvider = new GridPositionProvider(1.0);

// Wrap it to offset all generated points by the context's anchor
boolean isReversed = false;
PositionProvider anchoredGrid = new AnchorPositionProvider(gridProvider, isReversed);

// When invoked, the context will supply the anchor point
// For example, if anchor is (100, 50, 100), the grid will be generated
// relative to that point.
generatorContext.setAnchor(new Vector3d(100, 50, 100));
anchoredGrid.positionsIn(generatorContext);
```

### Anti-Patterns (Do NOT do this)
- **Null Context:** Passing a null `context` or a `context` with a null `anchor` will cause the provider to do nothing. The system is designed to fail silently in this case, which can mask configuration errors. Always ensure a valid anchor is present when using this provider.
- **Circular Wrapping:** Creating a circular dependency, such as wrapping a provider with an instance of itself, will result in a `StackOverflowError` during execution.
- **Misinterpreting Bounds:** The final bounds check (`VectorUtil.isInside`) is performed in the *original*, untranslated coordinate space. A child provider that generates points near the edge of its translated bounding box may have those points discarded if they fall outside the original parent bounds after being translated back.

## Data Pipeline
The AnchorPositionProvider acts as a transformation filter in the position generation pipeline. It intercepts the context, modifies it for its child, and then transforms the child's output before passing it downstream.

> Flow:
> Parent Generator -> `Context` (with original bounds & anchor) -> **AnchorPositionProvider** -> New `Context` (with translated bounds) -> Child `PositionProvider` -> Point (in translated space) -> **AnchorPositionProvider** (translates point back) -> Final Bounds Check -> Original `Consumer` -> Downstream System

