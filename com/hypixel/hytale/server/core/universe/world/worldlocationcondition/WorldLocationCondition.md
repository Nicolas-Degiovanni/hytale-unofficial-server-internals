---
description: Architectural reference for WorldLocationCondition
---

# WorldLocationCondition

**Package:** com.hypixel.hytale.server.core.universe.world.worldlocationcondition
**Type:** Polymorphic Strategy

## Definition
```java
// Signature
public abstract class WorldLocationCondition {
```

## Architecture & Concepts
WorldLocationCondition is an abstract base class that defines a contract for a server-side predicate function. Its primary role is to determine if a specific coordinate (x, y, z) within a World meets a set of predefined criteria. This class is a cornerstone of Hytale's data-driven world generation and gameplay logic systems.

The architecture is built around a polymorphic, serializable design pattern. The static CODEC field, an instance of CodecMapCodec, is the central mechanism. It allows the engine to deserialize various concrete implementations of WorldLocationCondition from data files (e.g., JSON, HOCON) based on a "Type" identifier. This enables designers and content creators to define complex spatial rules and triggers without modifying core engine code, significantly improving modularity and iteration speed.

Functionally, an instance of a WorldLocationCondition subclass represents a single, reusable rule, such as "Is this block exposed to the sky?" or "Is the biome at this location a desert?". These conditions can be composed into complex logical trees by higher-level systems.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly via constructors. They are instantiated by the Hytale serialization framework through the static CODEC field. This process is typically triggered when the server loads world generation assets, structure definitions, or quest data.
- **Scope:** These objects are typically transient and stateless. They are created, used to perform one or more tests, and then become eligible for garbage collection. They do not persist for the entire server session.
- **Destruction:** Managed entirely by the Java Garbage Collector. As these objects hold no persistent state or external resources, no explicit cleanup is required.

## Internal State & Concurrency
- **State:** The base class is stateless. Concrete implementations are designed to be **immutable**. Their configuration is fully determined at deserialization time and must not change during their lifetime. This immutability is critical for predictable behavior in a multi-threaded environment.
- **Thread Safety:** Instances are inherently thread-safe due to their immutable nature. The test method operates on a World object passed as a parameter, decoupling the condition's state from the world's state. The method itself is safe to call from any thread, provided the passed World object can be safely read from that thread.

**WARNING:** Developers creating subclasses of WorldLocationCondition MUST ensure they are fully immutable to maintain engine stability.

## API Surface
The public contract is minimal, enforcing the predicate pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(World, int, int, int) | boolean | Varies | The core method. Evaluates the condition at the given world coordinates. Complexity depends entirely on the subclass implementation. |
| equals(Object) | boolean | Varies | Abstract method forcing subclasses to provide value-based equality. |
| hashCode() | int | Varies | Abstract method forcing subclasses to provide a value-based hash code. |

## Integration Patterns

### Standard Usage
Direct interaction is uncommon. Typically, a higher-level game system retrieves a pre-configured condition and executes it.

```java
// A hypothetical structure placement service
// The 'placementCondition' would be deserialized from an asset file.
WorldLocationCondition placementCondition = structureDefinition.getPlacementCondition();

// Check if a structure can be placed at a candidate position
boolean canPlace = placementCondition.test(world, x, y, z);

if (canPlace) {
    // Proceed with placement logic
}
```

### Anti-Patterns (Do NOT do this)
- **Mutable Subclasses:** Implementing a subclass with mutable fields is a severe violation of the design contract. It introduces unpredictable behavior and race conditions, especially within the multi-threaded world generation pipeline.
- **Direct Instantiation:** Avoid calling constructors on concrete subclasses. All conditions should be defined in data files and loaded via the codec system to support the data-driven architecture.
- **Stateful Logic:** The test method should not modify the state of the World object. It is a read-only predicate.

## Data Pipeline
The flow of data and control for this component is central to its purpose. It acts as a bridge between static data definition and dynamic game logic.

> Flow:
> Game Asset (e.g., structure.json) -> Hytale Codec System -> **WorldLocationCondition** (Instance) -> World Generation Service -> boolean result -> Game Logic Execution

