---
description: Architectural reference for NPCReputationHolderSystem
---

# NPCReputationHolderSystem

**Package:** com.hypixel.hytale.builtin.adventure.npcreputation
**Type:** System Component

## Definition
```java
// Signature
public class NPCReputationHolderSystem extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
The NPCReputationHolderSystem is a reactive component system within Hytale's Entity Component System (ECS) framework. Its sole responsibility is to automatically assign a reputation affiliation to newly created Non-Player Characters (NPCs).

This system acts as a bridge between an entity's fundamental type definition (via the NPCEntity component) and the game's social reputation mechanics (via the ReputationGroupComponent). It operates by listening for the creation of any entity that has an NPCEntity component but does not yet have a ReputationGroupComponent.

Upon detecting such an entity, the system cross-references game asset data. It iterates through all defined ReputationGroup assets and checks if the new NPC's type belongs to any of the NPCGroup tags associated with that reputation. The first match found determines the NPC's faction, and the corresponding ReputationGroupComponent is attached to the entity. This automates the process of faction assignment, ensuring that game designers can define NPC affiliations declaratively in asset files without writing procedural code.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's core ECS framework during the world initialization sequence. Its dependencies, the ComponentType definitions, are injected by the framework at construction.
-   **Scope:** This system is a long-lived, stateless service. It persists for the entire duration of a running server session.
-   **Destruction:** The instance is discarded and garbage collected only when the server shuts down and its parent ECS container is dismantled.

## Internal State & Concurrency
-   **State:** This class is effectively immutable after construction. Its internal state consists of final references to component types and a pre-built Query object. It holds no per-entity state or mutable caches. All operations are performed using data passed into its methods.
-   **Thread Safety:** This system is **not thread-safe** and is designed to be operated exclusively by the main server game loop thread. The ECS framework guarantees that methods like onEntityAdd are called in a serialized, deterministic manner. Concurrent access from other threads would violate the ECS's transactional integrity and lead to data corruption.

## API Surface
The public API is intended for consumption by the ECS framework, not for direct use in game logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns the pre-built query that selects entities with an NPCEntity component but no ReputationGroupComponent. |
| onEntityAdd(holder, reason, store) | void | O(N*M) | Core logic. Triggered by the framework. Iterates through all N ReputationGroup assets and their M associated NPCGroup tags to find a match. Throws IllegalArgumentException on misconfigured asset data. |
| onEntityRemoved(holder, reason, store) | void | O(1) | No-op. This system does not react to entity removal. |

## Integration Patterns

### Standard Usage
A developer or content creator does not interact with this class directly. The system is triggered implicitly by the engine. To use it, one simply creates an entity and attaches an NPCEntity component to it. The framework ensures this system will then process the entity.

```java
// This is a conceptual example of what triggers the system.
// You would not write this code; the engine does it.

// 1. An entity is created in the world.
Holder<EntityStore> newEntity = world.createEntity();

// 2. An NPCEntity component is added. This makes the entity an NPC.
NPCEntity npcInfo = new NPCEntity(npcTypeIndex);
newEntity.addComponent(NPCEntity.TYPE, npcInfo);

// 3. The ECS framework detects the new entity matches the
//    NPCReputationHolderSystem's query and automatically calls
//    its onEntityAdd method, which may add a ReputationGroupComponent.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new NPCReputationHolderSystem()`. The ECS framework is responsible for its entire lifecycle. Manually creating an instance will result in a non-functional object that is not registered to receive engine events.
-   **Manual Invocation:** Do not call the onEntityAdd method directly. This bypasses the ECS query system and state management, which can lead to race conditions, duplicate components, or missed entity updates.
-   **Asset Misconfiguration:** The system's logic is critically dependent on the integrity of `ReputationGroup` and `NPCGroup` assets. If a `ReputationGroup` asset references a non-existent `npcGroup` string, the system will throw a fatal `IllegalArgumentException`, potentially halting server startup or causing a world chunk to fail loading.

## Data Pipeline
This system functions as a processor in a larger data flow for entity initialization. It enriches a basic entity by adding a component based on asset lookups.

> Flow:
> Entity Creation -> Add NPCEntity Component -> ECS Query Match -> **NPCReputationHolderSystem.onEntityAdd** -> Read Asset Data (ReputationGroup, NPCGroup) -> Add ReputationGroupComponent -> Entity is now affiliated with a faction

