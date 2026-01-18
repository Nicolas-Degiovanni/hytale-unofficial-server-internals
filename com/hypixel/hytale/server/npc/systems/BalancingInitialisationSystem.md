---
description: Architectural reference for BalancingInitialisationSystem
---

# BalancingInitialisationSystem

**Package:** com.hypixel.hytale.server.npc.systems
**Type:** Transient System Component

## Definition
```java
// Signature
public class BalancingInitialisationSystem extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
The BalancingInitialisationSystem is a specialized component system within the server-side Entity Component System (ECS) framework. Its primary function is to act as a bridge between an NPC's conceptual role definition and the concrete implementation of its in-game statistics.

This system operates on a specific subset of entities: those that possess both an NPCEntity component and an EntityStatMap component. When such an entity is added to the world, this system executes its logic. It reads the configured *initialMaxHealth* from the NPC's Role asset and applies this value to the entity's Health stat.

This is achieved by creating and adding a StaticModifier to the EntityStatMap. This modifier ensures the NPC's maximum health is precisely set at spawn time, overriding any default values from the base entity archetype. The system's explicit dependencies on RoleBuilderSystem and EntityStatsSystems.Setup guarantee a strict order of operations, preventing race conditions where this system might run before the NPC's role or base stats are available. It is a critical, one-shot initialization step in the NPC spawning pipeline.

### Lifecycle & Ownership
-   **Creation:** Instantiated automatically by the server's core ECS manager during the server bootstrap phase. The framework discovers all HolderSystem implementations and manages their lifecycle.
-   **Scope:** Session-scoped. A single instance of this system exists for the entire duration of the server's runtime.
-   **Destruction:** The instance is destroyed and eligible for garbage collection when the server shuts down and the parent ECS manager is dismantled.

## Internal State & Concurrency
-   **State:** This system is stateless. It holds immutable, pre-configured references to component types and dependency definitions. It does not store or cache any data related to the entities it processes; all state modifications are performed directly on the components of the target entity.
-   **Thread Safety:** Not thread-safe. This system is designed to be executed by a single-threaded game loop or a dedicated system-processing thread managed by the ECS framework. The framework's execution model guarantees that calls to onEntityAdd are serialized, preventing concurrent modification issues. **WARNING:** Manually invoking its methods from other threads will lead to data corruption and system instability.

## API Surface
The public contract is defined by its role as a HolderSystem, responding to framework callbacks rather than direct user invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDependencies() | Set | O(1) | Returns the set of systems that must execute before this one. Critical for ensuring initialization order. |
| getQuery() | Query | O(1) | Defines the entity archetype this system targets (must have NPCEntity and EntityStatMap). |
| onEntityAdd(holder, reason, store) | void | O(1) | Callback executed by the ECS framework when a matching entity is created. Contains the core health initialization logic. |
| onEntityRemoved(holder, reason, store) | void | O(1) | Callback executed on entity destruction. This implementation is a no-op. |

## Integration Patterns

### Standard Usage
A developer does not interact with this system directly. Its functionality is invoked implicitly by the engine when an entity matching its query is created. The correct "usage" is to ensure an NPC entity is properly configured with an NPCEntity component and an EntityStatMap.

```java
// PSEUDOCODE: How the engine implicitly uses this system
// 1. An entity is created with the required components.
Entity newNpc = world.createEntity();
newNpc.addComponent(new NPCEntity(someRole));
newNpc.addComponent(new EntityStatMap());

// 2. The ECS framework detects the new entity and, after satisfying
//    dependencies, invokes the system.
// BalancingInitialisationSystem.onEntityAdd(newNpc, ...);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new BalancingInitialisationSystem()`. The ECS framework is solely responsible for its lifecycle. Manually created instances will not be registered with the engine and will have no effect.
-   **Manual Invocation:** Never call `onEntityAdd` directly. Doing so bypasses the dependency resolution and state management of the ECS framework, which will almost certainly result in NullPointerExceptions or incorrect game state, as the required components may not have been initialized yet.

## Data Pipeline
This system is a key processing stage in the server-side NPC entity initialization sequence. Its position is strictly enforced by its declared dependencies.

> Flow:
> Entity Creation Event -> RoleBuilderSystem (Assigns Role) -> EntityStatsSystems.Setup (Creates base StatMap) -> **BalancingInitialisationSystem** (Reads Role health, applies a StaticModifier to the StatMap) -> NPC is fully initialized for gameplay.

---

