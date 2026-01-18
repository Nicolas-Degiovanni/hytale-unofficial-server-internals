---
description: Architectural reference for SensorCount
---

# SensorCount

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorCount extends SensorBase {
```

## Architecture & Concepts
The SensorCount component is a specialized predicate within the server-side NPC AI framework. Its primary function is to evaluate a specific world-state condition: whether the number of entities matching certain criteria within a defined spherical range falls between a minimum and maximum count.

Architecturally, SensorCount acts as a *query definition object*. It does not perform the spatial search or entity filtering itself. Instead, it encapsulates the parameters of the query (range, count, group filters) and delegates the execution to a more performant, centralized system: the PositionCache, which is attached to the NPC's Role.

This delegation pattern is critical. It decouples the AI behavior logic from the high-performance implementation of spatial queries. The SensorCount defines *what* to check, while the PositionCache determines *how* to check it efficiently, likely using spatial partitioning data structures. During its initialization, SensorCount "primes" the PositionCache by instructing it to begin tracking entities up to its maximum required range, ensuring the data is available and sorted when the `matches` method is eventually called during an AI tick.

## Lifecycle & Ownership
-   **Creation:** SensorCount instances are never created directly with `new`. They are instantiated exclusively by the BuilderSensorCount class during the server's NPC asset loading phase. This process is managed by the BuilderManager, which parses declarative AI behavior definitions (e.g., from JSON files) and constructs the corresponding component objects.
-   **Scope:** The object's lifetime is bound to the parent Role object that contains it. It is part of an NPC's static behavior definition and persists as long as that NPC's AI configuration is loaded in memory.
-   **Destruction:** The object is eligible for garbage collection when its parent Role is destroyed, typically when an NPC is unloaded or the server shuts down. It holds no persistent resources that require manual cleanup.

## Internal State & Concurrency
-   **State:** The component is **effectively immutable** after its initial lifecycle phase. All configuration fields (`minCount`, `maxRange`, `includeGroups`, etc.) are set in the constructor. A single field, `findPlayers`, is calculated and set during the `registerWithSupport` call. After this point, the object's state does not change.
-   **Thread Safety:** The `matches` method is conditionally thread-safe. It performs no writes to its own internal state, making it safe to call from any thread *as long as the provided Role and Store arguments are accessed in a thread-safe manner* according to the engine's main tick loop. The initial `registerWithSupport` method is a write operation and is not thread-safe; it must be called from the main server thread during the NPC initialization sequence.

## API Surface
The public API is minimal, designed for interaction with the core AI engine, not for general developer use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerWithSupport(Role role) | void | O(G) | **Lifecycle Method.** Initializes the sensor and primes the Role's PositionCache. G is the number of groups to check for the player tag. |
| matches(Ref, Role, double, Store) | boolean | O(N) | **Predicate Method.** Evaluates the condition. Delegates the spatial query to the Role's PositionCache. N is the number of entities within the sensor's range. |

## Integration Patterns

### Standard Usage
A developer does not interact with SensorCount directly. It is defined declaratively in an NPC asset file and invoked automatically by the NPC's behavior tree or state machine. The engine is responsible for calling `matches` during each relevant AI tick.

A conceptual asset definition might look like this:

```yaml
# Hypothetical NPC Behavior Asset
behaviors:
  - name: "FleeWhenCrowded"
    trigger:
      type: "SensorCount"
      range: [0.0, 10.0]  # 0 to 10 meters
      count: [5, 100]     # 5 to 100 entities
      includeGroups: ["HostileMonsters"]
    action: "ExecuteFleeBehavior"
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new SensorCount()`. The object is deeply tied to the asset loading pipeline and requires a Builder for correct initialization. Attempting to construct it manually will result in a non-functional component.
-   **Premature Evaluation:** Do not call the `matches` method on an instance before the engine has called `registerWithSupport`. Doing so will lead to incorrect results, as the `findPlayers` flag will be uninitialized and the underlying PositionCache will not have been instructed to track entities at the required range.

## Data Pipeline
SensorCount functions as a gate in a control flow rather than a step in a data transformation pipeline. Its output is a simple boolean that influences the NPC's decision-making process.

> Flow:
> AI Engine Tick -> Behavior Tree Node Evaluation -> **SensorCount.matches()** is called -> Query is delegated to PositionCache -> PositionCache iterates nearby entities -> **SensorCount.filterNPC()** is used as a predicate -> A final count is determined -> The count is compared to min/max -> **Boolean result** is returned -> Behavior Tree transitions state.

