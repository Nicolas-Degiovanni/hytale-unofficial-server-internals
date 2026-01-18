---
description: Architectural reference for RoleBuilderSystem
---

# RoleBuilderSystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** System Component

## Definition
```java
// Signature
public class RoleBuilderSystem extends HolderSystem<EntityStore> {
```

## Architecture & Concepts

The RoleBuilderSystem is a foundational component of the server-side Non-Player Character (NPC) framework. It functions as a one-shot factory and initializer within the Entity Component System (ECS). Its primary responsibility is to transform a newly spawned, skeletal NPC entity into a fully-featured, interactive agent.

This system acts as the critical bridge between an NPC's asset definition (the *Role*) and its live representation in the game world. When an entity is created with a minimal NPCEntity component (containing only a role name or index), this system is automatically triggered. It then orchestrates a complex build process, querying the central NPCPlugin for the corresponding role data, executing builder logic, and attaching dozens of required components such as AI behaviors, physical models, interaction handlers, and event listeners.

Architecturally, this system guarantees that all NPCs are constructed in a consistent, predictable, and data-driven manner. It decouples the act of spawning an NPC from the complex logic of its initialization. Other game systems can simply request the creation of an NPC by its role name, trusting that the RoleBuilderSystem will correctly assemble it.

## Lifecycle & Ownership

-   **Creation:** A single instance of RoleBuilderSystem is instantiated by the server's central ECS System Manager during the server bootstrap sequence. It is not created on-demand.
-   **Scope:** The instance is session-scoped. It persists for the entire lifetime of the server process and continuously monitors for new NPC entities.
-   **Destruction:** The system is de-registered and becomes eligible for garbage collection only when the server shuts down. It performs no explicit cleanup of its own.

## Internal State & Concurrency

-   **State:** The RoleBuilderSystem instance itself is effectively stateless. Its internal fields are final references to component types and dependency definitions, established at construction. All stateful operations are performed on the mutable Holder objects passed into its methods, never on the system object itself.

-   **Thread Safety:** **This class is not thread-safe.** As with most ECS systems, it is designed to be operated on by a single, main server thread that manages the EntityStore. All callbacks, such as onEntityAdd, are invoked serially by this thread. Any attempt to invoke its methods from a concurrent thread will result in race conditions, data corruption, and unpredictable server state.

## API Surface

The primary contract of this class is its implementation of the HolderSystem interface, not direct method invocation. Its behavior is triggered by the ECS framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, reason, store) | void | O(N) | Core logic. Triggered when a matching entity is added. Builds the NPC Role and attaches N components based on the Role's asset definition. |
| onEntityRemoved(holder, reason, store) | void | O(1) | No-op. This system performs no action on entity removal. |
| getQuery() | Query | O(1) | Defines the component signature for entities this system processes: must have NPCEntity and TransformComponent. |
| getDependencies() | Set | O(1) | Declares that this system must run *after* EntityStatsSystems and PhysicsValuesAddSystem. |

## Integration Patterns

### Standard Usage

Developers do not, and must not, interact with this system directly. The system is invoked automatically by the engine. The standard pattern is to create an entity and attach the NPCEntity component, which is sufficient to trigger this system's build process.

```java
// Correctly trigger the RoleBuilderSystem by creating a matching entity
EntityStore store = server.getWorld().getEntityStore();
Holder<EntityStore> newEntity = store.createEntity();

// 1. Add the essential components that match the system's query
newEntity.ensureComponent(TransformComponent.class);
NPCEntity npcInfo = new NPCEntity();

// 2. Specify the role by name. This is the input for RoleBuilderSystem.
npcInfo.setRoleName("hytale:guard");
newEntity.putComponent(NPCEntity.getComponentType(), npcInfo);

// The ECS framework will now automatically invoke RoleBuilderSystem.onEntityAdd for this entity.
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create an instance using `new RoleBuilderSystem()`. The ECS framework manages the singleton instance. Direct creation will result in a non-functional object that is not registered to receive entity events.
-   **Manual Invocation:** Do not call `onEntityAdd` manually. This bypasses critical ECS lifecycle management and state tracking, leading to partially initialized entities and severe bugs.
-   **Premature Component Access:** Do not attempt to access components that this system is responsible for creating (e.g., DisplayNameComponent, StateEvaluator) on the same tick that the NPC is spawned. The RoleBuilderSystem runs within the ECS update loop, and the components will not be present until its `onEntityAdd` method has completed.

## Data Pipeline

The RoleBuilderSystem is a key processing stage in the NPC entity initialization pipeline. It consumes a minimally defined entity and outputs a fully configured one.

> Flow:
> Spawner Logic or Command -> `EntityStore.createEntity()` -> Entity created with `NPCEntity` (containing role name) -> ECS framework matches entity to **RoleBuilderSystem** query -> `onEntityAdd` is invoked -> System reads role name -> System queries `NPCPlugin` for role asset and builder -> Builder logic executes -> **RoleBuilderSystem** attaches multiple new components (Model, AI, Interactions, etc.) to the entity -> Entity is now fully initialized and ready for processing by other systems (e.g., AI, Physics).

If the build process fails at any point, the pipeline is shunted to an error state.

> Failure Flow:
> ... -> Builder logic throws an error or finds invalid data -> **RoleBuilderSystem** enters `fail()` method -> All existing components are stripped from the entity -> A single `FailedSpawnComponent` is attached -> The entity becomes an inert "tombstone" and is ignored by subsequent game logic systems.

