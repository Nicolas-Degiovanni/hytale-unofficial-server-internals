---
description: Architectural reference for SensorEvent
---

# SensorEvent

**Package:** `com.hypixel.hytale.server.npc.corecomponents.world`
**Type:** Abstract Component

## Definition

```java
// Signature
public abstract class SensorEvent extends SensorBase {
```

## Architecture & Concepts

The SensorEvent class is an abstract base component within the Non-Player Character (NPC) Artificial Intelligence framework. It serves as a foundational template for creating sensors that detect specific, transient world events rather than continuously tracking environmental state. This component is a critical link between the game world's state and an NPC's behavioral decision-making logic, encapsulated within a `Role`.

Its primary function is to answer the question: "Has a relevant event occurred near me this tick?". It achieves this by searching for player or NPC entities that satisfy a specific, subclass-defined condition within a configured range.

The class employs a strategy pattern through the `EventSearchType` enum, allowing designers to configure the target acquisition priority (e.g., `PlayerFirst`, `NpcOnly`). This provides fine-grained control over NPC reactions, enabling behaviors that might prioritize player-initiated events over those from other NPCs, or vice-versa.

Upon successful detection, SensorEvent can perform two key actions:
1.  It caches the position of the detected entity via its internal `EntityPositionProvider`.
2.  It "locks on" to the target by storing a reference to it in the NPC's `MarkedEntitySupport` system, making the target easily accessible to other AI components like actions or goals.

## Lifecycle & Ownership

-   **Creation:** Instances of SensorEvent subclasses are not created directly via code. They are instantiated by the server's asset loading pipeline, specifically through a corresponding `BuilderSensorEvent` during the deserialization of an NPC's definition file (e.g., a JSON or HOCON asset). The `BuilderSupport` context provides necessary dependencies during this process.
-   **Scope:** A SensorEvent instance is owned by an NPC's `Role` component. Its lifetime is tightly coupled to the lifetime of the NPC entity it belongs to. It is a per-NPC-instance component, not a global or shared system.
-   **Destruction:** The object is eligible for garbage collection when the parent NPC entity and its associated `Role` are unloaded or destroyed. There are no explicit destruction or cleanup methods that need to be invoked.

## Internal State & Concurrency

-   **State:** SensorEvent is stateful.
    -   Configuration fields like `range`, `searchType`, and `lockOnTargetSlot` are set once at creation and are effectively immutable.
    -   The `positionProvider` field holds mutable state, caching the last detected target's reference and position. This state is cleared or updated each time the `matches` method is evaluated.
-   **Thread Safety:** This class is not thread-safe. It contains no internal locking mechanisms. It is designed to be accessed exclusively by the server's main entity update thread for a given NPC.

    **WARNING:** Concurrent modification of a SensorEvent instance from multiple threads will lead to race conditions and unpredictable AI behavior. All interactions must be synchronized by the calling system, which is typically the server's single-threaded tick loop for that entity.

## API Surface

The public contract is primarily for the NPC `Role` system, while the protected abstract contract is for developers creating new sensor types.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(N) | The primary evaluation method. Returns true if a valid event target is found. Complexity depends on the subclass implementation of target searching. |
| getSensorInfo() | InfoProvider | O(1) | Returns the internal provider that caches the last known target position. |
| getPlayerTarget(ref, store) | Ref<EntityStore> | *Abstract* | **Subclass Contract.** Must be implemented to define the logic for finding a relevant player-based event target. |
| getNpcTarget(ref, store) | Ref<EntityStore> | *Abstract* | **Subclass Contract.** Must be implemented to define the logic for finding a relevant NPC-based event target. |

## Integration Patterns

### Standard Usage

A developer does not call methods on this class directly. Instead, they create a concrete subclass and define the target acquisition logic. The game engine's AI system then invokes the `matches` method during the NPC's update cycle.

A typical implementation defines the specific conditions for what constitutes an "event".

```java
// Example of a concrete implementation for detecting a "shout" event
public class SensorShoutEvent extends SensorEvent {

    public SensorShoutEvent(BuilderSensorEvent builder, BuilderSupport support) {
        super(builder, support);
    }

    @Override
    @Nullable
    protected Ref<EntityStore> getPlayerTarget(@Nonnull Ref<EntityStore> self, @Nonnull Store<EntityStore> store) {
        // Logic to find a nearby player who has recently used the "shout" emote
        // Returns a reference to the player entity if found, otherwise null.
    }

    @Override
    @Nullable
    protected Ref<EntityStore> getNpcTarget(@Nonnull Ref<EntityStore> self, @Nonnull Store<EntityStore> store) {
        // Logic to find a nearby NPC who has recently sent a "shout" signal
        // Returns a reference to the NPC entity if found, otherwise null.
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** It is impossible to instantiate this class directly as it is abstract. Subclasses must not be instantiated with `new`; they must be configured in asset files to be loaded by the server's builder system.
-   **Expensive Searches:** Implementing `getPlayerTarget` or `getNpcTarget` with computationally expensive or unoptimized world queries will cause severe server performance degradation, as these methods are called frequently during an NPC's update tick.
-   **State Leakage:** Do not store state in a concrete subclass that persists between calls to `matches` unless explicitly intended. The `positionProvider` is designed to be the canonical state holder for the detected target.

## Data Pipeline

The SensorEvent acts as a filter and processor in the NPC's perception pipeline. It transforms raw world state into a qualified, actionable target for the AI.

> Flow:
> NPC Role Update Tick -> **SensorEvent.matches()** -> World Entity Query (`getPlayerTarget`/`getNpcTarget`) -> Internal State Update (`positionProvider`) -> Marked Entity System (`MarkedEntitySupport`) -> Downstream AI Components (Actions, Goals)

