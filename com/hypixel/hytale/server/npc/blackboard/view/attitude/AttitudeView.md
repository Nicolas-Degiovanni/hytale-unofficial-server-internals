---
description: Architectural reference for AttitudeView
---

# AttitudeView

**Package:** com.hypixel.hytale.server.npc.blackboard.view.attitude
**Type:** Transient

## Definition
```java
// Signature
public class AttitudeView extends PrioritisedProviderView<IAttitudeProvider, AttitudeView> {
```

## Architecture & Concepts

The AttitudeView is a specialized component within the server-side NPC Artificial Intelligence framework. It operates as a view model for an NPC's **Blackboard**, a centralized knowledge store for AI agents. Its sole responsibility is to resolve the attitude of one entity (the NPC) towards another target entity.

This class implements a **Chain of Responsibility** pattern by extending PrioritisedProviderView. It consults a series of registered IAttitudeProvider delegates in a strict priority order until one of them returns a definitive Attitude. This architecture provides a highly extensible system for defining complex social and factional relationships.

The default provider chain is registered during construction and establishes a clear hierarchy for attitude resolution:
1.  **Priority 0 (Highest):** A direct, programmatic override from the WorldSupport system. This allows for quest-specific or temporary attitude changes, suchas making a typically hostile creature friendly for a short time.
2.  **Priority 200:** The primary lookup via the global AttitudeMap. This is the data-driven layer where default faction relationships (e.g., "Zombies are hostile to Players") are defined.
3.  **Priority Integer.MAX_VALUE (Lowest):** A final fallback that returns a default attitude based on the target's fundamental type (Player or NPC). This ensures the system always returns a valid, predictable result.

This component is fundamental to NPC decision-making, directly influencing behaviors like target selection, aggression, and social interaction.

### Lifecycle & Ownership

-   **Creation:** AttitudeView instances are not created directly. They are instantiated and managed by an NPC's Blackboard component. A view is typically created on-demand when first requested for a specific World context via `Blackboard.getView`.

-   **Scope:** The lifecycle of an AttitudeView is tightly coupled to the World instance it was created for. It is a transient, cached object within the Blackboard. If the parent NPC entity moves to a different world, the existing AttitudeView becomes stale and a new one must be fetched.

-   **Destruction:** There is no explicit destruction logic within this class. Instances are eligible for garbage collection when the parent Blackboard is destroyed or when the World they are associated with is unloaded, as the Blackboard will no longer hold a reference to it.

## Internal State & Concurrency

-   **State:** The internal state of an AttitudeView is **effectively immutable** after construction. It holds a final reference to a World and a list of providers that is populated once in the constructor. The class itself does not cache attitude lookups; every call to getAttitude performs a fresh query through the provider chain.

-   **Thread Safety:** This class is **thread-safe for read operations**. It contains no mutable state and no internal locks. All context required for an attitude calculation is passed as method arguments. Callers must ensure that the provided arguments, particularly the ComponentAccessor and entity references, are accessed in a thread-safe manner according to the engine's broader concurrency model.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAttitude(ref, self, target, accessor) | Attitude | O(N) | Resolves the attitude towards a target. Iterates through N registered providers. Returns Attitude.NEUTRAL if no provider yields a result. |
| getUpdatedView(ref, accessor) | AttitudeView | O(1) | Returns a valid view for the entity's current world. **CRITICAL:** This must be called if an entity can change worlds to prevent using a stale view. |
| isOutdated(ref, store) | boolean | O(1) | Always returns false. This indicates the view's state does not decay over time; it only becomes invalid if the underlying world context changes. |

## Integration Patterns

### Standard Usage

The AttitudeView should always be retrieved from an entity's Blackboard resource. Before querying for an attitude, especially for entities that can traverse dimensions or worlds, use getUpdatedView to ensure the view is valid for the current context.

```java
// How a developer should normally use this
Blackboard blackboard = componentAccessor.getResource(Blackboard.getResourceType());
AttitudeView view = blackboard.getView(AttitudeView.class, selfRef, componentAccessor);

// Ensure the view is for the correct world before using it
AttitudeView currentView = view.getUpdatedView(selfRef, componentAccessor);

// Now, perform the query
Attitude attitudeToTarget = currentView.getAttitude(selfRef, selfRole, targetRef, componentAccessor);

if (attitudeToTarget == Attitude.HOSTILE) {
    // Initiate combat logic
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new AttitudeView(world)`. This bypasses the Blackboard's management and caching layer, leading to memory leaks and incorrect behavior. Always use `blackboard.getView()`.

-   **Ignoring Context Updates:** Failing to call getUpdatedView for an entity that has changed worlds will result in querying against the configuration of the *previous* world. This can lead to severe logic errors, such as an NPC retaining a friendly attitude from a hub world while in a hostile dungeon.

-   **Long-Term Caching of Results:** Do not cache the result of getAttitude for an extended period. Attitudes can be changed dynamically by game logic (e.g., via the Priority 0 provider). The view should be queried at the time a decision is being made to ensure the most current information is used.

## Data Pipeline

The flow of data for an attitude query is a clear, prioritized chain of lookups.

> Flow:
> AI Behavior Tree Query -> Blackboard.getView(AttitudeView.class) -> **AttitudeView.getAttitude()** -> [Provider 1: World Override?] -> [Provider 2: Global Faction Map?] -> [Provider 3: Default Fallback] -> Attitude Enum -> AI Behavior Tree Decision

