---
description: Architectural reference for FlockMembershipSystems
---

# FlockMembershipSystems

**Package:** com.hypixel.hytale.server.flock
**Type:** Utility

## Definition
```java
// Signature
public class FlockMembershipSystems {
```

## Architecture & Concepts
The **FlockMembershipSystems** class is a static container for the core business logic and Entity Component Systems (ECS) that govern the lifecycle of entity groups, known as "flocks". It is not a service or a component itself, but rather a collection of reactive systems and utility functions that maintain the integrity of flock-related data structures.

This class acts as the central authority for all flock membership changes. Its primary responsibility is to translate the addition or removal of a **FlockMembership** component on an entity into concrete changes within a corresponding **EntityGroup** component, which represents the flock itself. It manages leader election, validates membership eligibility, and handles the creation and dissolution of flock entities.

The architecture is fundamentally event-driven, with nested system classes that subscribe to specific ECS events:
- **Component Changes:** Systems react when a **FlockMembership** component is added, removed, or modified on any entity.
- **Entity Lifecycle:** Systems react when an entity with a **FlockMembership** component is loaded into or unloaded from the world.
- **Game Events:** Systems hook into the damage pipeline to aggregate combat data at the flock level.

This design decouples the *intent* to join a flock (adding a component) from the *implementation* of managing the group, ensuring that flock integrity is maintained transactionally through the ECS **CommandBuffer**.

### Lifecycle & Ownership
- **Creation:** The **FlockMembershipSystems** class is never instantiated. Its nested system classes (**EntityRef**, **RefChange**, etc.) are instantiated and registered by the server's ECS framework during the bootstrap sequence, likely within a dedicated **FlockPlugin** or **EntityModule**.
- **Scope:** The registered systems are singletons that persist for the entire server session. They are active and listening for events as long as the server is running.
- **Destruction:** The systems are deregistered and garbage collected during server shutdown when the ECS world is torn down.

## Internal State & Concurrency
- **State:** The **FlockMembershipSystems** class is entirely stateless. All state is managed externally within the ECS **Store** through components like **FlockMembership**, **Flock**, and **EntityGroup**. The nested systems may hold immutable references to component types, but they do not maintain mutable state across ticks.
- **Thread Safety:** This class and its systems are **not thread-safe** and must only be accessed from the main server thread that manages the ECS world. The systems rely heavily on the **CommandBuffer** pattern to defer mutations, preventing concurrent modification exceptions during system execution. Any direct, multi-threaded invocation of its methods or systems will lead to severe state corruption and server instability.

## API Surface
The primary public contract consists of static utility methods for high-level flock operations and the system classes which are integrated into the ECS engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canJoinFlock(ref, flockRef, store) | boolean | O(1) | Checks if an entity is eligible to join a given flock based on role and flock capacity. |
| join(ref, flockRef, store) | void | O(1) | Initiates the process of joining a flock by adding a **FlockMembership** component to an entity. |
| EntityRef | class | - | An ECS system that manages flock membership when entities are loaded or unloaded from the world. |
| RefChange | class | - | An ECS system that reacts to live changes (add/set/remove) of the **FlockMembership** component. |
| OnDamageDealt | class | - | An ECS system that listens for damage events to track damage inflicted by flock members. |
| OnDamageReceived | class | - | An ECS system that listens for damage events to track damage received by flock members. |
| NPCAddedFromWorldGen | class | - | A cleanup system that removes **FlockMembership** from newly generated NPCs to prevent state conflicts. |

## Integration Patterns

### Standard Usage
Developers should not interact with the system classes directly. The intended pattern is to manipulate the **FlockMembership** component on an entity. The registered systems will automatically detect the change and execute the necessary logic.

The static helper methods can be used by other game systems, such as AI behaviors, to query flock state or initiate a join operation.

```java
// Example: An AI behavior tree deciding to join a nearby flock
Ref<EntityStore> entityToJoin = ...;
Ref<EntityStore> targetFlock = ...;
Store<EntityStore> store = context.getStore();

if (FlockMembershipSystems.canJoinFlock(entityToJoin, targetFlock, store)) {
    // This triggers the RefChange system to handle the actual logic
    FlockMembershipSystems.join(entityToJoin, targetFlock, store);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct System Invocation:** Never call methods like **onEntityAdded** or **onComponentSet** directly. These are callbacks intended exclusively for the ECS engine. Bypassing the engine will break the **CommandBuffer** and cause immediate state corruption.
- **Manual Group Management:** Do not add or remove entities from the **EntityGroup** component directly. All membership changes must be driven by adding or removing the **FlockMembership** component to ensure that leader election and data synchronization logic in these systems is correctly executed.
- **Ignoring the Command Buffer:** When interacting with related components from within another system, always use the provided **CommandBuffer** to get components. Reading directly from the **Store** may yield stale data within the same tick.

## Data Pipeline
**FlockMembershipSystems** acts as a reactive processor in several key data flows. It does not originate data but rather transforms state in response to events.

> **Flow 1: Entity Joining a Flock (Live)**
> AI System -> `FlockMembershipSystems.join()` -> `store.putComponent(FlockMembership)` -> **RefChange System** -> `EntityGroup.add(ref)` -> Leader Election -> Chunk Marked Dirty

> **Flow 2: Entity Leaving a Flock (by Component Removal)**
> Game Logic -> `commandBuffer.removeComponent(FlockMembership)` -> **RefChange System** -> `EntityGroup.remove(ref)` -> New Leader Election -> Chunk Marked Dirty

> **Flow 3: Entity Loading from Storage**
> Chunk Load -> Entity Deserialization -> **EntityRef System** -> `joinOrCreateFlock()` -> `EntityGroup.add(ref)` -> State Reconciliation

> **Flow 4: Combat Data Aggregation**
> Damage Calculation -> `DamageModule` fires event -> **OnDamageDealt/OnDamageReceived Systems** -> `flockComponent.getNextDamageData().onInflictedDamage()` -> State updated on central **Flock** entity

