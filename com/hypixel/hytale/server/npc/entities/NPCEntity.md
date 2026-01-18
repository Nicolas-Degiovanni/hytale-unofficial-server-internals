---
description: Architectural reference for NPCEntity
---

# NPCEntity

**Package:** com.hypixel.hytale.server.npc.entities
**Type:** Component / Entity Data

## Definition
```java
// Signature
public class NPCEntity extends LivingEntity implements INonPlayerCharacter {
```

## Architecture & Concepts
The NPCEntity class is the foundational data component for all server-side Non-Player Characters. It does not contain behavior logic itself; instead, it serves as a stateful container within the server's Entity-Component-System (ECS) framework.

Its primary architectural role is to bridge an entity's core state (position, health, inventory) with its complex AI behaviors. This is achieved through a **Strategy Pattern**, where an NPCEntity is dynamically associated with a **Role** object. The NPCEntity holds the *data*, while the assigned Role implements the *behavior*, such as combat tactics, pathfinding decisions, and state transitions. This design effectively decouples the generic concept of an "NPC" from the specific logic of a "Zone 1 Wolf" or a "Town Guard".

NPCEntity is a critical participant in the server's AI ecosystem, integrating deeply with the **Blackboard** system. It functions as both a producer and consumer of world information:
*   **Producer:** It observes world events (e.g., a block being broken nearby) and publishes notifications to the Blackboard via methods like notifyBlockChange.
*   **Consumer:** It queries the Blackboard for spatial or state-based information (e.g., "what blocks are in this area?") to inform the Role's decision-making process.

Furthermore, the class is designed for persistence and network synchronization. The static CODEC field provides a robust serialization and deserialization contract, allowing NPC state to be saved to disk during world chunk unloading and restored upon reloading.

## Lifecycle & Ownership
-   **Creation:** NPCEntity instances are **never** created directly using the new keyword. They are instantiated by the server's entity management systems, primarily during world generation, player-triggered spawning events, or deserialization from world storage via the NPCEntity.CODEC. A Role must be assigned to the entity shortly after creation for it to function correctly.
-   **Scope:** An instance of NPCEntity exists for the entire lifetime of a specific non-player character in the game world. Its state persists as long as its containing world chunk is loaded in memory.
-   **Destruction:** Destruction is managed by the server's despawning logic. An NPC is marked for despawning by setting the isDespawning flag, which typically initiates a countdown or a despawn animation. The final removal of the entity and its components from the EntityStore is handled by the core ECS garbage collection process.

## Internal State & Concurrency
-   **State:** The NPCEntity is highly **mutable**. It represents the live, real-time state of a character in the world, including dynamic data such as its current target, pathfinding status via PathManager, active AI timers in AlarmStore, and temporary despawn counters. It also caches performance-critical data, such as the current movement speed multiplier derived from status effects.
-   **Thread Safety:** This class is **not thread-safe** and is designed for single-threaded access. All modifications and reads must occur on the main server game loop thread. Unsynchronized access from other threads will result in severe concurrency issues, including data corruption and server instability.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getRole() | Role | O(1) | Returns the behavior strategy object associated with this NPC. Returns null if not initialized. |
| setAppearance(ref, name, accessor) | boolean | O(N) | Changes the visual model of the NPC. Involves asset loading and component replacement. |
| playAnimation(ref, slot, id, accessor) | void | O(1) | Dispatches a request to the animation system to play a specific animation on the NPC's model. |
| notifyBlockChange(type, notification) | void | O(1) | Publishes a block-related event to the AI Blackboard system for other entities to observe. |
| notifyEntityEvent(type, notification) | void | O(1) | Publishes an entity-related event to the AI Blackboard system. |
| getBlockTypeBlackboardView(ref, store) | BlockTypeView | O(1) | Retrieves a view into the Blackboard for querying block data. Lazily initializes the view. |
| addReservation(playerUUID) | void | O(1) | Marks the NPC as "in use" by a player, preventing other players from initiating interactions. |
| removeReservation(playerUUID) | void | O(1) | Releases an interaction reservation on the NPC. |

## Integration Patterns

### Standard Usage
Interaction with an NPCEntity should always be performed through a valid entity reference and a ComponentAccessor, typically within the context of a System or a Role's update tick. The primary interaction point is often the Role, which encapsulates the high-level logic.

```java
// Within a server system, get the NPC component from an entity reference
Ref<EntityStore> npcRef = ...;
ComponentAccessor<EntityStore> accessor = ...;

NPCEntity npc = accessor.getComponent(npcRef, NPCEntity.getComponentType());
if (npc != null && npc.getRole() != null) {
    // Delegate a high-level action to the NPC's Role
    npc.getRole().onAttackedBy(npcRef, attackerRef, accessor);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call new NPCEntity(). The ECS and spawning systems are solely responsible for creating and managing entity lifecycles. Direct creation bypasses world registration and will result in a non-functional, phantom entity.
-   **State without Role:** An NPCEntity without an assigned Role is an incomplete and inert object. The Role must be set during the initialization sequence for the NPC to have any behavior.
-   **Asynchronous Modification:** Do not access or modify an NPCEntity from any thread other than the main server thread. All interactions must be scheduled and executed as part of the server's tick cycle.
-   **Direct Blackboard Manipulation:** Do not attempt to modify the internal blackboard fields directly. Always use the provided API, such as notifyBlockChange or getBlockTypeBlackboardView, to ensure proper integration with the AI eventing and caching systems.

## Data Pipeline
The NPCEntity acts as a central hub for data flowing to and from the AI systems.

**Inbound Data Flow (Sensing the World):**
> Flow:
> World Event (e.g., Block Break) -> Server Event Bus -> Blackboard System -> Blackboard View -> **NPCEntity** -> Role (AI Decision)

**Outbound Data Flow (Acting on the World):**
> Flow:
> Role (AI Decision) -> **NPCEntity** (e.g., calls playAnimation, setTarget) -> Animation / Combat / Movement Systems -> World State Change

**Persistence Flow:**
> Flow:
> **NPCEntity** (Live State) -> NPCEntity.CODEC -> Serialized Binary Data -> World Storage (Disk)

