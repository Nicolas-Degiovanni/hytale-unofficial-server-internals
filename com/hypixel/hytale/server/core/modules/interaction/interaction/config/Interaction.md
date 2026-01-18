---
description: Architectural reference for the Interaction class, the core abstraction for all game actions.
---

# Interaction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config
**Type:** Polymorphic Asset Type

## Definition
```java
// Signature
public abstract class Interaction
   implements Operation,
   JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, Interaction>>,
   NetworkSerializable<com.hypixel.hytale.protocol.Interaction> {
```

## Architecture & Concepts

The Interaction class is the fundamental abstraction for all discrete, stateful actions an entity can perform within the game world. It serves as the server-side authority for behaviors such as swinging a weapon, casting a spell, using an item, or blocking. This class is not a concrete action itself, but rather a base template from which all specific interactions (e.g., MeleeAttackInteraction, PlaceBlockInteraction) inherit.

Its design is deeply rooted in a **data-driven architecture**. Concrete interactions are defined as JSON assets, which are deserialized at runtime into instances of Interaction subclasses. This allows designers and developers to create or modify complex game mechanics without changing core engine code. The static CODEC and ABSTRACT_CODEC fields are central to this pattern, defining the schema for how JSON configuration is mapped to object properties.

The Interaction system is tightly integrated with several core engine modules:

*   **Asset System:** Interactions are assets managed by the global AssetStore. They are identified by a unique string ID and are loaded on demand.
*   **Entity Component System (ECS):** An Interaction operates on entities via a CommandBuffer and direct component access (e.g., TransformComponent, Player). It does not hold entity state itself; instead, it receives an InteractionContext containing references to the acting entity and its components.
*   **Networking:** As a NetworkSerializable object, an Interaction's configuration can be packaged and sent to clients. The `handle` method is responsible for broadcasting the effects of an interaction (via PlayInteractionFor packets) to nearby players, ensuring visual and auditory consistency across clients.
*   **Interaction Chains:** Interactions are rarely executed in isolation. They are typically composed into a sequence of operations within an InteractionChain. This allows for complex, multi-stage actions where one Interaction can lead to another.

## Lifecycle & Ownership

-   **Creation:** Interaction objects are **never instantiated directly** using the *new* keyword in game logic. They are deserialized from JSON asset files by the Hytale AssetStore during server initialization or when asset packs are loaded. The static `getAssetStore` method provides access to the central repository where all Interaction assets are managed.
-   **Scope:** An Interaction asset, once loaded, is effectively a singleton for its specific type (e.g., there is only one instance of the "hytale:sword_swing" Interaction). It persists for the entire server session and is shared across all executions of that action.
-   **Destruction:** The object is garbage collected when the AssetStore is cleared, which typically only occurs during a full server shutdown. The `cachedPacket` field uses a SoftReference, allowing the garbage collector to reclaim memory from the serialized network packet under memory pressure.

## Internal State & Concurrency

-   **State:** An Interaction object should be considered **immutable after initialization**. Its fields (runTime, viewDistance, rules, etc.) represent static configuration loaded from JSON. All mutable, per-execution state (e.g., current progress, target entity) is stored externally in the InteractionContext object that is passed into its methods. This design is critical, as a single Interaction asset is shared and executed concurrently for many different entities.

-   **Thread Safety:** The object is inherently thread-safe for read operations due to its immutable configuration. Execution methods like `tick` and `simulateTick` are designed to be called from a single, well-defined context, such as the main server thread for a given entity. State is isolated to the InteractionContext, preventing race conditions between different entities performing the same action. The use of `SpatialResource.getThreadLocalReferenceList` in `sendPlayInteract` further demonstrates a design conscious of multi-threaded environments by avoiding shared collections.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(...) | final void | O(C) | The primary execution method. Advances the state of the interaction for one server tick. Contains the core logic for the action's duration, conditions, and effects. |
| simulateTick(...) | final void | O(C) | A parallel method to `tick` used for simulation or predictive logic without applying final side effects. |
| handle(...) | void | O(N) | Manages the network replication of the interaction. Broadcasts packets to N nearby clients to trigger visual and audio effects. |
| compile(...) | void | O(1) | A configuration-time method used to add this interaction as an operation to an OperationsBuilder, typically for constructing an InteractionChain. |
| toPacket() | com.hypixel.hytale.protocol.Interaction | O(1) | Serializes the interaction's configuration data into a network packet. The result is cached in a SoftReference for performance. |
| getAssetStore() | static AssetStore | O(1) | Provides global access to the central registry of all loaded Interaction assets. |

## Integration Patterns

### Standard Usage

An Interaction is never retrieved or executed directly. It is orchestrated by higher-level systems like the InteractionManager, which manages the lifecycle of an InteractionChain for a given entity.

```java
// Logic within a system like InteractionManager
// 1. Identify the interaction asset (e.g., from an item)
Interaction interaction = Interaction.getAssetMap().getAsset("hytale:fireball");

// 2. Create a context for the execution
InteractionContext context = new InteractionContext(entityRef, ...);

// 3. In the entity's update loop, tick the interaction
// This is a simplified representation; a full InteractionChain would manage this.
interaction.tick(entityRef, entity, isFirstRun, deltaTime, type, context, cooldownHandler);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new SomeInteraction()`. This bypasses the asset system entirely, resulting in an unconfigured and non-functional object that is not registered with the engine.
-   **State Modification:** Do not modify the fields of a loaded Interaction at runtime. These objects are shared, and changing a property like `runTime` would affect every entity performing that action across the entire server.
-   **External Tick Management:** Do not call `tick` outside of the context of an InteractionChain or a similar managing system. The state (firstRun, time) and context must be carefully managed to ensure correct behavior.

## Data Pipeline

The flow of data during the execution of a single interaction follows a clear, server-authoritative pattern.

> Flow:
> Player Input Packet -> **InteractionManager** (Server) -> Identifies Interaction Asset -> Creates **InteractionChain** -> **Interaction.tick()** (executes logic, updates InteractionContext) -> **Interaction.handle()** (broadcasts effects) -> **PlayInteractionFor Packet** -> Nearby Clients (play VFX/SFX)

