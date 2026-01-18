---
description: Architectural reference for TargetMemory
---

# TargetMemory

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.memory
**Type:** Component (Data)

## Definition
```java
// Signature
public class TargetMemory implements Component<EntityStore> {
```

## Architecture & Concepts
The TargetMemory component is a fundamental data structure within the server-side NPC AI framework. It functions as the short-term combat memory for a single NPC entity, enabling stateful and reactive behaviors.

As a component in Hytale's Entity Component System (ECS), TargetMemory strictly adheres to the principle of separating data from logic. It holds the *state* of what an NPC is aware of—specifically friendly and hostile entities—but contains no behavioral logic itself. The logic that populates, prunes, and interprets this data resides in dedicated AI *Systems*, such as the NPCCombatActionEvaluator.

This component is attached to an NPC entity to grant it the capability of tracking other entities over time. It uses performance-oriented collections from the fastutil library to minimize memory overhead and avoid Java's object boxing for primitive types, which is critical for a system that may manage thousands of NPCs simultaneously.

The core state is divided into two primary categories:
*   **Known Hostiles:** Entities that pose a threat.
*   **Known Friendlies:** Entities that are allied or neutral.

This separation is the basis for all higher-level tactical decisions, such as target prioritization, flanking maneuvers, and protective actions.

## Lifecycle & Ownership
- **Creation:** TargetMemory instances are not instantiated directly using the *new* keyword in game logic. They are created by the engine's entity management system when an NPC is spawned. Typically, this component is defined as part of an entity's archetype and is instantiated via its `clone()` method to create a clean memory slate for the new NPC.

- **Scope:** The lifecycle of a TargetMemory component is strictly bound to the lifecycle of the parent NPC entity to which it is attached. It persists as long as the NPC exists in the world.

- **Destruction:** The component is marked for garbage collection and its state is lost when its parent NPC entity is destroyed, despawned, or otherwise removed from the world. No manual cleanup is required by game logic developers.

## Internal State & Concurrency
- **State:** The internal state of TargetMemory is highly **mutable**. Its collections are designed to be continuously updated by AI sensory and evaluation systems during each server tick. It effectively caches the NPC's perception of its immediate surroundings. The `rememberFor` field is immutable after construction and defines the time-to-live for memories.

- **Thread Safety:** This component is **not thread-safe** and must be considered thread-hostile. In the context of the Hytale server, all interactions with an entity's components, including TargetMemory, must be performed on the main server thread or the specific world thread responsible for that entity.

    **WARNING:** Unsynchronized access from other threads (e.g., networking, physics callbacks) will lead to `ConcurrentModificationException`, data corruption, and unpredictable AI behavior. All cross-thread interactions must be marshaled into the main game loop as scheduled tasks.

## API Surface
The public API provides direct, high-performance access to the underlying memory collections for AI Systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getKnownFriendlies() | Int2FloatOpenHashMap | O(1) | Returns the raw map of friendly entity IDs to a floating-point value, typically a timestamp of last-seen time. |
| getKnownFriendliesList() | List | O(1) | Returns the list of direct references to known friendly entities. |
| getKnownHostiles() | Int2FloatOpenHashMap | O(1) | Returns the raw map of hostile entity IDs to their last-seen timestamp. |
| getKnownHostilesList() | List | O(1) | Returns the list of direct references to known hostile entities. |
| getClosestHostile() | Ref | O(1) | Retrieves the cached reference to the current primary target. May be null. |
| setClosestHostile(ref) | void | O(1) | Sets the primary target. This is typically called by a target selection system. |
| clone() | Component | O(1) | Creates a new, empty TargetMemory instance with the same `rememberFor` configuration. Used by the ECS framework. |

## Integration Patterns

### Standard Usage
TargetMemory is designed to be retrieved from an entity within an AI System. The system then populates the memory based on sensory input (e.g., vision checks) and reads from it to inform decision-making.

```java
// Within an AI System processing a specific entity...
TargetMemory memory = entity.getComponent(TargetMemory.class);

if (memory == null) {
    // This entity does not have combat memory capabilities.
    return;
}

// 1. A sensory system updates the memory with new sightings.
List<Entity> visibleHostiles = findVisibleHostiles(entity);
for (Entity hostile : visibleHostiles) {
    memory.getKnownHostiles().put(hostile.getId(), world.getTime());
}

// 2. A decision system uses the memory to select an action.
Ref<EntityStore> primaryTarget = memory.getClosestHostile();
if (primaryTarget != null) {
    // ...initiate attack action...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new TargetMemory()`. Components are managed exclusively by the entity framework. Manually created instances will not be attached to any entity or processed by any systems.

- **Cross-Thread Modification:** Never modify the collections within TargetMemory from an asynchronous task or a different thread. This will break the server's tick synchronization guarantees.

- **Long-Lived References:** Avoid caching a reference to a TargetMemory component outside the scope of a single method or tick. The parent entity could be destroyed at any moment, making the cached reference a memory leak or a source of null pointer exceptions. Always re-fetch the component from the entity at the start of a processing cycle.

## Data Pipeline
TargetMemory acts as a stateful repository in the AI data flow. It does not process data itself but serves as a critical junction between perception and action.

> Flow:
> AI Sensory System (Vision, Hearing) -> **TargetMemory Component** (Write State) -> AI Target Selection System -> **TargetMemory Component** (Read/Write State) -> AI Action Evaluation System -> **TargetMemory Component** (Read State) -> Selected Combat Action

