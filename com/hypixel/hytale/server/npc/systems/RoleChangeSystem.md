---
description: Architectural reference for RoleChangeSystem
---

# RoleChangeSystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** System Component

## Definition
```java
// Signature
public class RoleChangeSystem extends TickingSystem<EntityStore> {
```

## Architecture & Concepts

The RoleChangeSystem is a critical component of the server-side NPC framework, responsible for managing the fundamental behavioral state of an NPC, known as its *Role*. A Role dictates an NPC's components, AI logic, and capabilities. Changing a Role is a destructive and complex operation, and this system provides a robust, asynchronous, and decoupled mechanism to handle it.

Architecturally, this system functions as a deferred command processor. Instead of allowing other game systems to directly manipulate an NPC's components to change its Role—a process prone to error and race conditions—it exposes a static request method. Systems that need to alter an NPC's Role submit a `RoleChangeRequest` to a global queue. The RoleChangeSystem then processes this queue once per server tick.

This asynchronous, queue-based pattern ensures that all major entity restructurings occur at a controlled point in the game loop, preventing state corruption and simplifying the logic for the calling systems. The core implementation strategy is **destructive and reconstructive**: for each request, the system completely removes the target NPC entity from the world, strips it of all Role-specific components, re-configures its core NPCEntity component with the new Role data, and then re-adds the entity to the world. This guarantees a clean state transition without the complexity of component-by-component reconciliation.

The system's execution order is explicitly defined to run *after* the NewSpawnStartTickingSystem, ensuring that newly created NPCs are fully initialized before any role change requests are processed for them within the same tick.

### Lifecycle & Ownership
- **Creation:** Instantiated by the server's Entity Component System (ECS) framework during the loading phase of the NPCPlugin. Its dependencies, such as required ComponentType and ResourceType definitions, are provided via constructor injection.
- **Scope:** Singleton per `EntityStore`. An instance of this system persists for the entire lifetime of the world or server session it is associated with.
- **Destruction:** The system is destroyed and garbage collected when its parent `EntityStore` is unloaded, typically during a server shutdown or world change.

## Internal State & Concurrency
- **State:** The RoleChangeSystem instance itself is effectively stateless. Its member fields are final, immutable references to ECS type definitions. The operational state it manages—the queue of pending role changes—is stored externally within the `RoleChangeQueue` resource, which is globally accessible within the `EntityStore`.
- **Thread Safety:** **This system is not thread-safe and must be operated exclusively from the main server thread.** The `tick` method is invoked by the single-threaded ECS engine. The static `requestRoleChange` method accesses a shared, mutable `ArrayDeque`, which is not a concurrent collection. Calling `requestRoleChange` from an asynchronous task or different thread will lead to a `ConcurrentModificationException` or other unpredictable state corruption. All interactions must be synchronized with the main game loop.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, systemIndex, store) | void | O(N) | Processes all pending requests in the `RoleChangeQueue`. N is the number of requests. Called automatically by the ECS engine. |
| requestRoleChange(...) | static void | O(1) | Submits a request to change an NPC's role. This is the primary public entry point for other systems. |

## Integration Patterns

### Standard Usage
The intended pattern is for other game logic, such as a quest script or an AI behavior tree, to invoke the static `requestRoleChange` method. The caller provides the entity reference and the new role details, and the system handles the complex transition on a subsequent tick.

```java
// Example: A quest system triggers a role change for an NPC
// to turn them from a "Villager" to a "Guard".

// Assume 'npcRef' is a valid Ref<EntityStore> to the NPC
// Assume 'guardRole' is the target Role object
// Assume 'store' is the active EntityStore

// Request the change. The system will handle the rest.
RoleChangeSystem.requestRoleChange(
    npcRef,
    guardRole,

    // The integer index for the new role
    guardRole.getIndex(),

    // true to update the NPC's visual appearance
    true,

    // The initial state and sub-state in the new role's state machine
    "PATROLLING",
    null,
    store
);
```

### Anti-Patterns (Do NOT do this)
- **Asynchronous Invocation:** Do not call `requestRoleChange` from a separate thread. The underlying queue is not thread-safe. All requests must originate from the main server thread.
- **Direct Instantiation:** Do not create an instance of RoleChangeSystem using `new`. The ECS framework is responsible for its creation and lifecycle management.
- **Manual Ticking:** Never call the `tick` method directly. It is designed to be driven exclusively by the ECS engine to maintain correct execution order and state.
- **Direct Queue Manipulation:** Avoid getting the `RoleChangeQueue` resource and modifying it directly. Always use the `requestRoleChange` static helper method to ensure requests are formatted correctly.

## Data Pipeline
The flow of data for a role change operation is linear and centrally processed, ensuring stability.

> Flow:
> External Logic (e.g., Quest System, AI Behavior) -> `RoleChangeSystem.requestRoleChange()` -> **RoleChangeQueue** (Resource) -> **RoleChangeSystem.tick()** -> Entity is removed from `EntityStore` -> Components are stripped -> NPCEntity is reconfigured -> Entity is re-added to `EntityStore` -> NPC operates with new Role.

