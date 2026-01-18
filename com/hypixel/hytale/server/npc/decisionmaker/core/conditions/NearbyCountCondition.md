---
description: Architectural reference for NearbyCountCondition
---

# NearbyCountCondition

**Package:** com.hypixel.hytale.server.npc.decisionmaker.core.conditions
**Type:** Component / Configuration

## Definition
```java
// Signature
public class NearbyCountCondition extends ScaledCurveCondition {
```

## Architecture & Concepts

The **NearbyCountCondition** is a specific implementation of a Scorer within the server-side NPC Utility AI framework. It is not a standalone service but rather a configurable data-driven component that contributes to an NPC's decision-making process. Its primary function is to calculate a raw numerical input—the count of nearby entities belonging to a specified **NPCGroup**—which is then transformed into a final utility score by its parent class, **ScaledCurveCondition**.

This condition is a critical building block for creating emergent group behaviors such as swarming, retreating when outnumbered, or seeking out allies.

Architecturally, it serves as a bridge between the high-level AI decision-making layer and the lower-level spatial query system. It achieves high performance by offloading the expensive work of spatial partitioning and distance checking to the **PositionCache** component, which is attached to each NPC's **Role**. During an initialization phase, this condition "registers its interest" in entities within a given range, allowing the **PositionCache** to optimize its internal data structures and pre-calculate relevant entity sets.

## Lifecycle & Ownership

-   **Creation:** Instances of **NearbyCountCondition** are not created programmatically via a constructor. They are deserialized from NPC behavior asset files (e.g., JSON) by the server's **BuilderManager** using the provided static **CODEC**. This process occurs when the server loads its NPC configurations at startup or during a hot-reload.

-   **Scope:** The object's lifetime is bound to the loaded NPC behavior asset. It exists as part of an in-memory representation of an NPC's potential behaviors. A single instance may be shared by all NPCs of the same type.

-   **Destruction:** The object is marked for garbage collection when the server unloads or reloads the corresponding NPC behavior assets. There is no manual destruction method.

## Internal State & Concurrency

-   **State:** The state of this object is defined by its configuration fields: *range*, *npcGroup*, and the derived *npcGroupIndex* and *includePlayers* flags. This state is populated once during deserialization and the subsequent *setupNPC* call. After this initialization phase, the object is **effectively immutable**. Its internal state is not modified during evaluation.

-   **Thread Safety:** This class is **not thread-safe** and is not designed for concurrent access. All interactions, from initialization via *setupNPC* to evaluation via *getInput*, are expected to occur on the primary thread responsible for the owning NPC's AI update tick. The use of a **CommandBuffer** in the *getInput* signature indicates that any world-modifying actions resulting from this condition's evaluation are deferred and executed in a synchronized manner by the core engine.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setupNPC(Role role) | void | O(1) | Initializes the condition for a specific NPC. Critically, this primes the NPC's **PositionCache** to begin tracking entities within the configured range. **WARNING:** This must be called before any evaluation. |
| getInput(...) | double | O(k) | Calculates the raw input value (the entity count). Complexity is O(k), where k is the number of entities within the pre-filtered set provided by **PositionCache**, not the total number of entities in the world. |

## Integration Patterns

### Standard Usage

This component is not intended to be used directly in Java code. Instead, it is configured declaratively within an NPC behavior asset file. The engine's deserialization and AI systems handle its lifecycle and invocation.

*Example NPC Behavior Snippet (Conceptual JSON)*
```json
{
  "scorers": [
    {
      "type": "NearbyCountCondition",
      "Range": 20.0,
      "NPCGroup": "Hostile.Zombies",
      "curve": { ... }
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new NearbyCountCondition()`. The object will be uninitialized and will cause a **NullPointerException** when used. Always define it in a data asset.
-   **Premature Evaluation:** Do not attempt to call *getInput* before the engine has called *setupNPC* for the relevant NPC **Role**. Doing so will bypass the **PositionCache** optimizations and may result in incorrect data or exceptions.
-   **Excessive Range:** Configuring an extremely large *range* value can severely degrade server performance. It forces the **PositionCache** to track a much larger set of entities for every NPC using this condition, increasing memory usage and CPU load during the spatial query phase.

## Data Pipeline

The flow of data for this condition begins with asset loading and culminates in a utility score used for AI decision-making.

> Flow:
> NPC Behavior Asset -> Server **BuilderManager** -> **NearbyCountCondition** Instance -> Engine calls `setupNPC` -> **PositionCache** is primed -> AI Update Tick -> Engine calls `getInput` -> Count is returned -> **ScaledCurveCondition** maps count to utility score -> AI Decision Maker selects action

