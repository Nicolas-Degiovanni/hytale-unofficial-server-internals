---
description: Architectural reference for ClientSourcedSelector
---

# ClientSourcedSelector

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.selector
**Type:** Transient

## Definition
```java
// Signature
@Deprecated
public class ClientSourcedSelector implements Selector {
```

## Architecture & Concepts

**WARNING: This class is deprecated.** It represents an outdated and potentially insecure design pattern. New development should use server-authoritative selection mechanisms.

The ClientSourcedSelector is a specialized implementation of the Selector interface. Its primary architectural function is to override server-side hit detection and instead trust the client's report of which entities were targeted. This component acts as a **Decorator** around another Selector instance, a pattern evident in its constructor which accepts a parent Selector.

It delegates most operations, such as block selection and ticking, to its parent. However, it completely replaces the logic for entity selection. When `selectTargetEntities` is called, this class does not perform any raycasting or volume queries. Instead, it reads a pre-populated list of hits directly from the `InteractionContext`, which was filled by the network layer from an incoming client packet.

This design prioritizes client-side responsiveness over server authority, a trade-off that is generally discouraged for cheat-sensitive game logic. The server's only validation is to confirm that the network ID reported by the client corresponds to a valid entity in the world. It does not validate line-of-sight, range, or any other interaction constraints.

## Lifecycle & Ownership

-   **Creation:** Instantiated by the server's interaction configuration system when an interaction is defined to use a client-sourced strategy. It is created with a reference to a parent Selector and the specific `InteractionContext` of the interacting entity.
-   **Scope:** The object's lifetime is tightly coupled to the interaction instance it serves. It is a short-lived, transient object, not a global service.
-   **Destruction:** It is eligible for garbage collection as soon as the interaction configuration is no longer active or the entity performing the interaction is removed from the world.

## Internal State & Concurrency

-   **State:** The ClientSourcedSelector is effectively immutable. Its internal fields (`parent`, `context`) are final and set during construction. However, it is a consumer of external, mutable state provided by the `InteractionContext`, specifically the `hitEntities` array.
-   **Thread Safety:** This class is **not thread-safe** and must only be used from the main server game loop thread. Its methods operate on the `CommandBuffer` and `EntityStore`, which are core engine components that assume single-threaded access within a given tick. The `InteractionContext` it reads from may be populated by a network thread, but access within the selector's methods is assumed to be synchronized by the overarching game loop.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(commandBuffer, ref, time, runTime) | void | O(N) | Delegates the tick operation to the parent Selector. N is the complexity of the parent's tick method. |
| selectTargetEntities(commandBuffer, ref, consumer, filter) | void | O(H) | **Overrides parent.** Iterates through client-reported hits. H is the number of hits. Ignores the `filter` predicate entirely. |
| selectTargetBlocks(commandBuffer, ref, consumer) | void | O(M) | Delegates the block selection operation to the parent Selector. M is the complexity of the parent's method. |

## Integration Patterns

### Standard Usage

**NOTE:** The standard usage is to **avoid this class**. The following example is for historical and educational purposes to understand how it would have been integrated by the interaction system. A developer would not call this directly.

```java
// CONCEPTUAL: How the interaction system might use the selector
// This is NOT a pattern to be replicated.

// 1. The system receives client hit data and populates the context
InteractionContext context = entity.getInteractionContext();
context.getClientState().hitEntities = ... // Data from network packet

// 2. The system invokes the selector during the interaction phase
Selector selector = new ClientSourcedSelector(new DefaultSelector(), context);
selector.selectTargetEntities(cmdBuffer, entityRef, (targetRef, hitLocation) -> {
    // Apply game logic to the client-reported target
    applyDamage(targetRef, 10.0);
});
```

### Anti-Patterns (Do NOT do this)

-   **Use in New Code:** The primary anti-pattern is using this class for any new feature. Its deprecated status indicates it is obsolete.
-   **Security-Sensitive Logic:** Never use this selector for interactions that have a meaningful impact on game state, such as damage, resource gathering, or quest progression. It is highly vulnerable to client-side cheats.
-   **Ignoring the Filter:** Developers relying on the `filter` predicate passed to `selectTargetEntities` will find it is completely ignored by this implementation, which can lead to unexpected behavior and bugs.

## Data Pipeline

The data flow for this component originates from the game client, making it fundamentally different from server-authoritative selectors.

> Flow:
> Client-Side Hit Detection -> Network Packet (SelectedHitEntity array) -> Server Protocol Layer -> `InteractionContext.clientState` -> **ClientSourcedSelector.selectTargetEntities** -> Interaction Logic (Consumer) -> World State Change

