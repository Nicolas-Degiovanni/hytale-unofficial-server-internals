---
description: Architectural reference for InteractionView
---

# InteractionView

**Package:** com.hypixel.hytale.server.npc.blackboard.view.interaction
**Type:** Scoped Service

## Definition
```java
// Signature
public class InteractionView extends PrioritisedProviderView<ReservationProvider, InteractionView> {
```

## Architecture & Concepts
The InteractionView is a specialized component within the server-side NPC Blackboard system. Its primary architectural function is to centralize and arbitrate the logic that determines if an NPC is available for player interaction. It acts as the definitive source of truth for an NPC's "reservation" status.

This class implements the **Prioritised Provider** pattern by extending PrioritisedProviderView. It maintains an ordered list of ReservationProvider delegates. When a query is made, it iterates through these providers from highest to lowest priority, returning the first definitive reservation status it receives. This design allows for a highly extensible system where different game mechanics (e.g., quests, combat states, cinematic sequences) can register their own reservation logic without modifying core NPC code.

A default, lowest-priority provider is registered at creation time. This provider directly checks the reservation state on the NPCEntity component, serving as a final fallback for the most basic reservation checks.

## Lifecycle & Ownership
- **Creation:** InteractionView instances are not created directly. They are instantiated and managed by a parent Blackboard resource. A system requests a view of this type from the Blackboard, which acts as a factory and cache. An instance is typically created on the first request for a given World.

- **Scope:** The lifecycle of an InteractionView is tightly coupled to a specific World instance. It holds a direct reference to the World it services and is considered invalid if the context switches to a different world.

- **Destruction:** The component is eligible for garbage collection when its parent Blackboard is cleared or the associated World is unloaded from memory. The `cleanup` and `onWorldRemoved` methods, while currently empty, are the designated hooks for explicit resource release.

## Internal State & Concurrency
- **State:** This class is **stateful and mutable**. It maintains an internal reference to a World and a collection of registered ReservationProvider instances inherited from its parent. This collection can be modified at runtime by other systems registering new providers.

- **Thread Safety:** This component is **not thread-safe**. It is designed to be accessed exclusively from the main server thread responsible for its parent World's tick cycle. Unsynchronized, concurrent access to its provider list or internal state will lead to undefined behavior, including ConcurrentModificationExceptions.

    **WARNING:** All interactions with an InteractionView instance must be synchronized with the owning world's update loop.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getReservationStatus(npcRef, playerRef, accessor) | ReservationStatus | O(N) | Queries all registered providers to determine the NPC's interaction reservation status. N is the number of providers. |
| getUpdatedView(ref, accessor) | InteractionView | O(1) | Ensures the view is valid for the current world context. Returns a new instance from the Blackboard if the world has changed. |

## Integration Patterns

### Standard Usage
The correct pattern is to retrieve the view from the NPC's Blackboard resource within the current execution context. This ensures you have the correct, world-specific instance.

```java
// Correctly retrieve the view from the Blackboard resource
Blackboard blackboard = componentAccessor.getResource(Blackboard.getResourceType());
InteractionView view = blackboard.getView(InteractionView.class, npcRef, componentAccessor);

// Query the reservation status
ReservationStatus status = view.getReservationStatus(npcRef, playerRef, componentAccessor);

if (status == ReservationStatus.NOT_RESERVED) {
    // Proceed with interaction
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new InteractionView(world)`. This creates a rogue instance disconnected from the Blackboard's provider registry. Such an instance will only contain the default reservation logic and will ignore all custom game-system providers, leading to inconsistent behavior.

- **Instance Caching:** Do not cache and reuse an InteractionView instance across different scopes or ticks without validation. Always call `getUpdatedView` to ensure the instance is still valid for the current world context, as the underlying world may have changed.

- **Cross-Thread Access:** Never access an InteractionView from an asynchronous task or a different thread. All calls must originate from the owning World's main thread to prevent race conditions.

## Data Pipeline
The InteractionView serves as a decision point in the player-to-NPC interaction flow. Data flows from a player action, through the view's logic, and results in a status that gates the subsequent game logic.

> Flow:
> Player Interaction Attempt -> Server Interaction System -> Blackboard Resource Lookup -> **InteractionView::getReservationStatus** -> Iteration over ReservationProviders -> Final ReservationStatus -> Allow/Deny Interaction Logic

