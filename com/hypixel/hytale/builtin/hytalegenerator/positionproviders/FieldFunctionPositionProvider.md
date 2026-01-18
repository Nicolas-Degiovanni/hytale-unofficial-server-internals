---
description: Architectural reference for FieldFunctionPositionProvider
---

# FieldFunctionPositionProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.positionproviders
**Type:** Transient

## Definition
```java
// Signature
public class FieldFunctionPositionProvider extends PositionProvider {
```

## Architecture & Concepts
The FieldFunctionPositionProvider is a specialized filter component within the world generation pipeline. It operates based on the **Decorator Pattern**, wrapping another source PositionProvider to selectively pass through or discard 3D world positions.

Its primary function is to enable conditional placement of world features based on environmental data. It achieves this by coupling a stream of candidate positions from its wrapped provider with a **Density** function (e.g., a noise field representing temperature, moisture, or elevation).

For each position supplied by the source provider, this class samples the value of the Density field at that exact location. It then compares this sampled value against a list of pre-configured numeric ranges, known as **Delimiters**. If the value falls within any of the valid delimiter ranges, the position is accepted and forwarded to the next stage of the generation pipeline. If it falls outside all ranges, the position is discarded.

This mechanism is fundamental for creating ecologically plausible environments, such as restricting certain plant types to specific altitudes or ensuring ore veins only appear in regions of high geological stress represented by a density field.

### Lifecycle & Ownership
- **Creation:** Instantiated and configured by a higher-level world generation system, typically when defining a specific placement rule for an object or feature. It is not a globally shared service.
- **Scope:** The lifetime of a FieldFunctionPositionProvider instance is typically confined to the execution of a single world generation pass for a specific zone or chunk.
- **Destruction:** The object is eligible for garbage collection once the generation pass that created it completes and all references to it are dropped. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** This class is stateful. Its behavior is defined by its internal `field`, `positionProvider`, and `delimiters` list. The `delimiters` list is mutable and is populated via the `addDelimiter` method after construction. The core dependencies (`field`, `positionProvider`) are immutable after construction.

- **Thread Safety:** This class is **not thread-safe**. The internal `delimiters` list is a standard, non-synchronized ArrayList.
    - **WARNING:** Modifying the delimiter list via `addDelimiter` while the `positionsIn` method is being executed by any thread will result in undefined behavior, likely a ConcurrentModificationException.
    - The `positionsIn` method is designed to be executed within a single worker thread as part of a larger, thread-confined generation task. All configuration should be completed before the instance is used in a parallelized generation context.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| FieldFunctionPositionProvider(field, positionProvider) | constructor | O(1) | Constructs the provider, injecting the Density field and the source PositionProvider to wrap. |
| addDelimiter(min, max) | void | O(1) amortized | Adds a valid numeric range `[min, max)` to the filter criteria. Must be called before execution. |
| positionsIn(context) | void | O(N * D) | Executes the filtering pipeline. N is the number of positions from the source provider; D is the number of delimiters. |

## Integration Patterns

### Standard Usage
The intended usage is to chain providers together. The FieldFunctionPositionProvider is configured with its filter ranges and then used to process the output of another provider, such as one that generates positions in a grid or sphere.

```java
// 1. Define the source of candidate positions (e.g., a grid)
PositionProvider sourcePositions = new GridPositionProvider(...);

// 2. Define the environmental data field (e.g., moisture noise)
Density moistureField = new SimplexNoiseDensity(...);

// 3. Create the filter, wrapping the source provider
FieldFunctionPositionProvider filter = new FieldFunctionPositionProvider(moistureField, sourcePositions);

// 4. Configure the filter to only accept positions where moisture is high
filter.addDelimiter(0.7, 0.9);

// 5. Execute the pipeline. The consumer will only receive positions
// that are on the grid AND have a high moisture value.
filter.positionsIn(generationContext);
```

### Anti-Patterns (Do NOT do this)
- **Execution Before Configuration:** Calling `positionsIn` before adding any delimiters via `addDelimiter` is valid but useless. The filter will have no valid ranges and will discard all incoming positions.
- **Concurrent Modification:** Do not call `addDelimiter` from one thread while `positionsIn` is being executed on another. The object's state must be considered frozen once processing begins.
- **State Reuse Across Threads:** Do not share a single instance of this class across multiple, independent worker threads if those threads could potentially modify its state. Each generation task should configure its own instance.

## Data Pipeline
This class acts as a conditional gate in a data stream. It intercepts positions, evaluates them, and either forwards them or terminates their path.

> Flow:
> Source PositionProvider -> Raw Position Stream -> **FieldFunctionPositionProvider** (Sample Density & Compare to Delimiters) -> Filtered Position Stream -> Context Consumer

