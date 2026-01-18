---
description: Architectural reference for WorldLocationProvider
---

# WorldLocationProvider

**Package:** com.hypixel.hytale.builtin.adventure.objectives.config.worldlocationproviders
**Type:** Polymorphic Strategy

## Definition
```java
// Signature
public abstract class WorldLocationProvider {
```

## Architecture & Concepts
The WorldLocationProvider is an abstract base class that forms the core of a data-driven strategy pattern. Its primary role within the Adventure Mode objective system is to define a contract for algorithms that find a specific coordinate (a Vector3i) within a game world based on a set of conditions.

The key architectural feature is its deep integration with the Hytale serialization framework via the static **CODEC** field, which is a CodecMapCodec. This mechanism allows game designers to declaratively specify complex location-finding logic in configuration files (e.g., JSON or HOCON) without writing new Java code.

During asset loading, the game engine deserializes an objective's configuration. When it encounters a WorldLocationProvider definition, it reads a "Type" field (e.g., "LocationRadius") and uses the static CODEC registry to find and instantiate the corresponding concrete implementation (e.g., LocationRadiusProvider). This makes the system highly extensible, as new location-finding strategies can be added simply by creating a new subclass and registering it in the static initializer.

### Lifecycle & Ownership
- **Creation:** Instances are not created directly via a constructor. They are instantiated exclusively by the Hytale Codec system during the deserialization of a parent configuration object, such as an adventure objective or a procedural generation rule. The specific subclass created is determined by the "Type" field in the source data.
- **Scope:** Transient and owned. The lifetime of a WorldLocationProvider instance is strictly tied to the parent configuration object that contains it. They are effectively immutable value objects representing a piece of game logic.
- **Destruction:** Instances are managed by the Java garbage collector. They are deallocated when their parent configuration object is unloaded or no longer referenced. No manual cleanup is required.

## Internal State & Concurrency
- **State:** The base class is stateless. Concrete implementations are designed to be **immutable**. Their parameters, such as a search radius or a block tag, are injected by the codec system upon creation and must not change during the object's lifetime.
- **Thread Safety:** Instances are inherently **thread-safe** due to their immutability. The primary `runCondition` method is a pure function with respect to the provider's state. It can be safely called from multiple threads, provided that access to the passed-in World object is properly synchronized by the calling system.

## API Surface
The public contract is minimal, exposing only the core execution logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| runCondition(World, Vector3i) | Vector3i | Implementation-Dependent | Executes the location-finding logic. Returns a valid Vector3i on success or null if no suitable location is found. The complexity varies significantly by implementation (e.g., a simple check is O(1), a radius scan could be O(n³)). |

## Integration Patterns

### Standard Usage
A developer or game designer does not interact with this class directly in imperative code. Instead, it is defined declaratively within a larger configuration file. A higher-level system, such as an ObjectiveManager, then uses the deserialized object to perform the check.

**Conceptual Configuration (e.g., JSON)**
```json
{
  "objectiveName": "Find The Ancient Forge",
  "trigger": {
    "type": "ENTER_ZONE",
    "locationProvider": {
      "Type": "TagBlockHeight",
      "tag": "hytale:ancient_forge_marker",
      "radius": 64,
      "minHeight": 40,
      "maxHeight": 80
    }
  }
}
```

**Engine-Side Execution**
```java
// Somewhere inside the objective processing logic...
// 'provider' is the deserialized WorldLocationProvider object from the config
Vector3i targetLocation = provider.runCondition(world, player.getPosition());

if (targetLocation != null) {
    // A valid location was found, proceed with objective logic
    world.spawnBeaconAt(targetLocation);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create instances with `new`. The system's extensibility and data-driven nature rely entirely on the codec for instantiation. Direct creation bypasses this and will lead to non-functional or brittle game logic.
- **Stateful Implementations:** Do not create subclasses that modify their internal state after construction. This violates the immutability contract and will cause severe, difficult-to-debug concurrency issues if the objective is processed by multiple threads.

## Data Pipeline
The WorldLocationProvider acts as a configurable logic component within a larger data-processing flow, transforming a declarative rule into an executable action.

> Flow:
> Adventure Objective Config File -> Hytale Codec Engine -> **WorldLocationProvider Instance** -> Objective Execution System -> `runCondition()` -> World State Query -> Result (Vector3i or null)

