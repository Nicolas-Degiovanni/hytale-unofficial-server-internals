---
description: Architectural reference for OffsetPositionProvider
---

# OffsetPositionProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.positionproviders
**Type:** Transient

## Definition
```java
// Signature
public class OffsetPositionProvider extends PositionProvider {
```

## Architecture & Concepts
The OffsetPositionProvider is a concrete implementation of the **Decorator** design pattern, designed to augment the functionality of another PositionProvider. Its sole purpose is to wrap an existing provider and apply a fixed spatial translation to all positions it generates.

This class is a fundamental component within the world generation pipeline, acting as a coordinate transformation layer. The operational logic is inverted from what one might expect:
1.  A query is received for positions within a specific bounding box (the context window).
2.  The OffsetPositionProvider translates this *query window* in the *opposite* direction of its configured offset.
3.  It delegates the query to the wrapped PositionProvider, using the newly translated window.
4.  As the wrapped provider yields positions, the OffsetPositionProvider intercepts each one and translates it *forward* by the original offset before passing it to the final consumer.

This mechanism allows generator logic to be defined in a local coordinate space (e.g., relative to 0,0,0) and then be stamped into the world at any desired location simply by wrapping it with an OffsetPositionProvider.

## Lifecycle & Ownership
-   **Creation:** Instantiated directly via its constructor, typically during the assembly of a world generator's configuration. It is composed by passing another PositionProvider instance that it will decorate.
-   **Scope:** The lifetime of an OffsetPositionProvider is tied to the world generator configuration that created it. It is a lightweight, value-like object with no external dependencies beyond the provider it wraps.
-   **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and is eligible for collection as soon as its parent configuration is no longer referenced.

## Internal State & Concurrency
-   **State:** **Immutable**. The internal state, consisting of the offset vector and the reference to the wrapped provider, is established at construction and cannot be modified thereafter. The constructors defensively clone the incoming offset vectors to guarantee this immutability and prevent external mutation.
-   **Thread Safety:** **Conditionally Thread-Safe**. This class contains no mutable state or synchronization primitives. Its thread safety is therefore entirely inherited from the wrapped PositionProvider. Given that world generation is a highly parallelized process, it is a design assumption that the providers it wraps are themselves thread-safe.

## API Surface
The public contract is minimal, consisting only of the inherited method from the PositionProvider base class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| positionsIn(Context) | void | O(N) | Translates the query window, delegates to the wrapped provider, and translates the resulting positions. N is the number of positions returned by the wrapped provider. |

## Integration Patterns

### Standard Usage
The canonical use case is to displace the output of another provider. This is essential for placing pre-defined features or structures within the world.

```java
// A provider that generates positions for a feature at the world origin
PositionProvider featureProvider = new MyFeaturePositionProvider();

// We want to place this feature at world coordinates (512, 64, 1024)
Vector3i worldOffset = new Vector3i(512, 64, 1024);

// Wrap the base provider to create a displaced version
PositionProvider placedFeatureProvider = new OffsetPositionProvider(worldOffset, featureProvider);

// Querying the new provider will now yield positions relative to the offset
// For example, a position of (1, 2, 3) from the original provider will become
// (513, 66, 1027).
placedFeatureProvider.positionsIn(worldGenerationContext);
```

### Anti-Patterns (Do NOT do this)
-   **External Vector Mutation:** Do not retain a reference to the vector passed into the constructor with the intent of modifying it later. The provider clones the vector on instantiation, and any subsequent changes to the original vector will have no effect. This will lead to non-deterministic behavior that is difficult to debug.
-   **Excessive Chaining:** While it is possible to chain multiple OffsetPositionProviders, doing so can degrade floating-point precision and make spatial reasoning unnecessarily complex. When multiple translations are needed, it is architecturally cleaner to compute a single final offset and apply it with one provider.

## Data Pipeline
The OffsetPositionProvider acts as a two-way transformation stage in the data flow. It transforms the input query and then transforms the output data.

> Flow:
> Incoming Context (Window A) -> **OffsetPositionProvider** (Translates window to A - offset) -> Wrapped Provider -> Position (p) -> **OffsetPositionProvider** (Translates position to p + offset) -> Outgoing Consumer

