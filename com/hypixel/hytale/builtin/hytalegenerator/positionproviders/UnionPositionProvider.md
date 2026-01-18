---
description: Architectural reference for UnionPositionProvider
---

# UnionPositionProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.positionproviders
**Type:** Composite Component

## Definition
```java
// Signature
public class UnionPositionProvider extends PositionProvider {
```

## Architecture & Concepts
The UnionPositionProvider is a structural component that implements the **Composite design pattern**. It does not generate any positions on its own. Instead, its sole function is to act as a container for a collection of other PositionProvider instances, treating the entire group as a single, unified provider.

This class is a fundamental building block for creating complex and layered world generation rules. It allows developers to combine multiple, simpler position generation strategies into a single, cohesive operation. For example, a world feature might need to be placed according to a grid in some areas and randomly in others. A UnionPositionProvider can contain both a GridPositionProvider and a RandomPositionProvider, executing them sequentially within the same generation pass.

It functions as a multiplexer or a delegator. When its `positionsIn` method is invoked, it iterates through its internal list of child providers and calls the `positionsIn` method on each one, passing along the same Context object. The order of execution is determined by the order of providers in the list supplied during construction.

### Lifecycle & Ownership
- **Creation:** Instances are typically created by a world generation configuration loader. They are materialized when a generator preset defines a list or a union of position sources. Direct, manual instantiation is rare and usually confined to unit tests or highly specific procedural code.
- **Scope:** Transient. The lifetime of a UnionPositionProvider is strictly bound to the execution of a single world generation task. It is created, used once to populate a Context, and then becomes eligible for garbage collection.
- **Destruction:** Managed by the Java Garbage Collector. There are no native resources or explicit cleanup methods. Its memory is reclaimed once the parent generator that created it completes its work and releases its reference.

## Internal State & Concurrency
- **State:** The internal state is the list of child PositionProviders. This list is populated at construction time and is **effectively immutable** thereafter. The `final` keyword on the field prevents the list reference from being reassigned, and the class exposes no methods for adding, removing, or reordering providers post-construction.

- **Thread Safety:** This component is **conditionally thread-safe**. The class itself introduces no concurrency hazards; its own state is immutable and the iteration logic is safe. However, its overall thread safety is entirely dependent on the components it interacts with:
    1.  **Child Providers:** If any of the contained PositionProvider instances are not thread-safe, then the UnionPositionProvider is not thread-safe when used in a multi-threaded context.
    2.  **Context Object:** The `PositionProvider.Context` object passed to `positionsIn` must be designed for concurrent access if multiple threads are to operate on it.

    **Warning:** Do not assume this class is thread-safe. Its safety is inherited from the child providers it contains. All providers in the collection must be thread-safe for the union to be considered safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| UnionPositionProvider(List) | Constructor | O(N) | Constructs the provider, copying N references from the input list. |
| positionsIn(Context) | void | O(N) | Sequentially invokes `positionsIn` on N contained providers. The overall complexity is the sum of the complexities of all child providers. |

## Integration Patterns

### Standard Usage
The standard pattern is to define a collection of providers and pass them to the constructor. This is typically handled by a configuration system that deserializes world generation presets.

```java
// Example: Combining a grid and a random scatter for feature placement
List<PositionProvider> providers = new ArrayList<>();
providers.add(new GridPositionProvider(/* config */));
providers.add(new RandomScatterPositionProvider(/* config */));

// The UnionPositionProvider executes them in the order they were added
PositionProvider combinedProvider = new UnionPositionProvider(providers);

// The context will first be populated by the grid, then by the scatter
combinedProvider.positionsIn(generationContext);
```

### Anti-Patterns (Do NOT do this)
- **Recursive Nesting:** Avoid deeply nesting UnionPositionProviders within other UnionPositionProviders. While functionally valid, this can create performance bottlenecks due to excessive iteration and method call overhead. It also makes debugging world generation logic exceptionally difficult. Flatten provider lists where possible.
- **Empty Provider List:** Instantiating a UnionPositionProvider with an empty list is a logical error. It performs no work and adds unnecessary overhead. Configuration loaders and procedural logic should validate and prune such cases.
- **Ignoring Execution Order:** The providers are executed in the order they appear in the constructor's list. Do not assume the execution is parallel or unordered. If one provider's output is meant to influence a subsequent one, the order is critical.

## Data Pipeline
The UnionPositionProvider acts as a sequential processing node. It does not transform data but rather orchestrates the flow of the Context object to a series of sub-processors.

> Flow:
> World Generator -> `positionsIn(Context)` -> **UnionPositionProvider** -> `Child1.positionsIn(Context)` -> `Child2.positionsIn(Context)` -> ... -> Modified Context returned to World Generator

