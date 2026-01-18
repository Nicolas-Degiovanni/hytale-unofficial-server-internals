---
description: Architectural reference for Interactions
---

# Interactions

**Package:** com.hypixel.hytale.server.core.modules.interaction
**Type:** Component

## Definition
```java
// Signature
public class Interactions implements Component<EntityStore> {
```

## Architecture & Concepts
The Interactions class is a data-holding **Component** within Hytale's server-side Entity-Component-System (ECS) architecture. It is not a service or manager; its sole responsibility is to attach a collection of interaction behaviors to a game entity.

This component acts as a data contract, defining *what* can be done with an entity. It maps an enumeration of interaction types, such as PRIMARY_ACTION, to a string identifier. This identifier is a key used by the server's scripting or game logic engine to execute a specific behavior when a player interacts with the entity. For example, it can associate an NPC's "use" action with a script path like `scripts/village/blacksmith/on_trade`.

A critical architectural feature is the `isNetworkOutdated` dirty flag. This flag is central to the engine's network replication strategy. Any mutation of the component's state sets this flag to true, signaling to the network synchronization system that this component's data must be sent to clients during the next replication tick.

## Lifecycle & Ownership
-   **Creation:** An Interactions component is never instantiated directly. It is created and attached to an Entity by the `EntityStore` or a higher-level game system, typically during entity spawning or as a result of game logic (e.g., a quest adding a new interaction option to an NPC).
-   **Scope:** The lifecycle of an Interactions component is strictly bound to its parent entity. It persists only as long as the entity exists in the world.
-   **Destruction:** The component is marked for garbage collection when its parent entity is removed from the `EntityStore`. There is no manual destruction method.

## Internal State & Concurrency
-   **State:** The component's state is highly mutable. Its core data consists of a map of `InteractionType` to `String` identifiers and an optional `interactionHint`. All public mutator methods modify this internal state.
-   **Thread Safety:** **This component is not thread-safe.** It is designed to be owned and modified exclusively by the main server game loop thread. Access from asynchronous tasks or other threads will lead to severe concurrency issues, including data corruption and missed network updates. All modifications must be synchronized with the server tick.

## API Surface
The public API is designed for state mutation and retrieval by server-side game logic and the core networking engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setInteractionId(type, id) | void | O(1) | Binds an interaction type to a logic identifier. Marks the component as dirty for network replication. |
| getInteractionId(type) | String | O(1) | Retrieves the identifier for a specific interaction type. Returns null if not present. |
| getInteractions() | Map | O(1) | Returns a **read-only** view of the entire interaction map. |
| setInteractionHint(hint) | void | O(1) | Sets a user-facing hint string (e.g., "Talk"). Marks the component as dirty. |
| consumeNetworkOutdated() | boolean | O(1) | Atomically reads and resets the network dirty flag. **Warning:** For engine-internal use only. |

## Integration Patterns

### Standard Usage
Game logic should retrieve the component from an entity and call its mutator methods to alter the entity's interactive behaviors.

```java
// Correctly modifying an entity's interactions from a server system
// Assume 'targetEntity' is a valid, loaded entity.

Interactions interactions = targetEntity.getComponent(Interactions.getComponentType());

if (interactions != null) {
    // Change the primary action to trigger a quest script
    interactions.setInteractionId(InteractionType.PRIMARY_ACTION, "hytale:quests/main/start_dialogue");

    // Update the UI hint for the player
    interactions.setInteractionHint("Speak with Guard");
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new Interactions()`. Components are managed by the ECS framework. Direct instantiation creates an orphaned object that is not attached to an entity and will not be processed by any engine systems.
-   **Manual Flag Management:** Do not call `consumeNetworkOutdated` from game logic. This method is the exclusive responsibility of the network replication system. Calling it manually will cause state changes to be lost and never sent to clients, resulting in desynchronization.
-   **Caching the Interactions Map:** Do not retrieve the interactions map and store a reference to it. The internal state can change at any time. Always request the component from the entity when you need to access its current state.

## Data Pipeline
The Interactions component is a source of state data that is consumed by the network engine. It does not process data itself.

> Flow:
> Server Game Logic -> `setInteractionId()` -> **Interactions Component State Change** -> Network Replication System reads `consumeNetworkOutdated()` -> Network Packet Serialization -> Client-side Entity Update

