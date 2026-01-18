---
description: Architectural reference for RootDependency
---

# RootDependency

**Package:** com.hypixel.hytale.component.dependency
**Type:** Utility / Singleton Factory

## Definition
```java
// Signature
public class RootDependency<ECS_TYPE> extends Dependency<ECS_TYPE> {
```

## Architecture & Concepts
The RootDependency class is a specialized, foundational component within the Entity-Component-System (ECS) framework. It does not represent a dependency on another system, but rather a dependency on the conceptual **root** of the execution graph. Its primary function is to anchor an ISystem to the absolute beginning or end of the system update order.

During the engine's startup phase, a DependencyGraph of all registered ISystem instances is constructed. This graph is then topologically sorted to produce a linear execution order for each tick. RootDependency provides the fixed entry and exit points required for this sort to function deterministically.

The class provides two globally accessible singletons:
- **FIRST:** Marks a system to be executed as early as possible.
- **LAST:** Marks a system to be executed as late as possible.

By design, a RootDependency is always an **Order.AFTER** dependency. A system can only declare that it runs *after* the conceptual start point or *after* all other systems that run before the end point.

### Lifecycle & Ownership
- **Creation:** The primary instances, FIRST and LAST, are static singletons instantiated by the JVM during class loading. They are not intended to be created by consumer code.
- **Scope:** As static singletons, these objects persist for the entire lifetime of the application.
- **Destruction:** The instances are garbage collected by the JVM when the application terminates and the class loader is unloaded.

## Internal State & Concurrency
- **State:** The RootDependency objects, particularly the static FIRST and LAST singletons, are **immutable**. Their internal state, including order and priority, is set upon instantiation and never changes.
- **Thread Safety:** This class is inherently thread-safe. Its immutable nature guarantees that concurrent reads from multiple threads during dependency graph resolution are safe and will not lead to race conditions or inconsistent state.

## API Surface
The primary API consists of static factory methods for retrieving the singleton instances.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| first() | RootDependency | O(1) | Returns the canonical singleton instance representing the earliest execution point. |
| last() | RootDependency | O(1) | Returns the canonical singleton instance representing the latest execution point. |
| firstSet() | Set | O(1) | Returns an immutable Set containing only the FIRST singleton. For convenience. |
| lastSet() | Set | O(1) | Returns an immutable Set containing only the LAST singleton. For convenience. |
| resolveGraphEdge(...) | void | O(1) | **Internal Contract.** Called by the ComponentRegistry. Adds an edge from the graph's root node to the target system, effectively anchoring it. |

## Integration Patterns

### Standard Usage
A system declares its position in the execution order by overriding its dependency-providing method and returning a set containing one of the RootDependency singletons. This is the sole intended use case.

```java
// An ISystem that must process network input before any other game logic.
public class NetworkInputSystem implements ISystem<ClientECS> {

    @Override
    @Nonnull
    public Set<Dependency<ClientECS>> getDependencies() {
        // By returning firstSet(), this system is guaranteed to be placed
        // at or near the beginning of the execution order for each tick.
        return RootDependency.firstSet();
    }

    // ... other system methods
}

// An ISystem that must render UI after all other game logic and simulation.
public class UIRenderingSystem implements ISystem<ClientECS> {

    @Override
    @Nonnull
    public Set<Dependency<ClientECS>> getDependencies() {
        // This system will be placed at the end of the execution order.
        return RootDependency.lastSet();
    }

    // ... other system methods
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new RootDependency()`. While the constructor is public, the framework is designed around the identity of the static `first()` and `last()` singletons. Creating new instances is unnecessary and bypasses the clear intent of the API.
- **Unsupported Order:** The class internally enforces an `Order.AFTER` relationship. Attempting to create a "before the root" dependency is conceptually invalid and will throw an UnsupportedOperationException during graph resolution.

## Data Pipeline
RootDependency does not process runtime data. Instead, it is a configuration object used during the engine's initialization pipeline to structure the execution flow of other systems.

> **Graph Construction Flow:**
> ISystem Definition -> ComponentRegistry Registration -> **RootDependency.resolveGraphEdge** -> DependencyGraph Edge Creation -> Topological Sort -> Final System Execution Order

