---
description: Architectural reference for CraftRecipeEvent
---

# CraftRecipeEvent

**Package:** com.hypixel.hytale.server.core.event.events.ecs
**Type:** Transient

## Definition
```java
// Signature
public abstract class CraftRecipeEvent extends CancellableEcsEvent {

    // Nested Event for Pre-Crafting Validation
    public static final class Pre extends CraftRecipeEvent {
        public Pre(@Nonnull CraftingRecipe craftedRecipe, int quantity) {
            super(craftedRecipe, quantity);
        }
    }

    // Nested Event for Post-Crafting Notification
    public static final class Post extends CraftRecipeEvent {
        public Post(@Nonnull CraftingRecipe craftedRecipe, int quantity) {
            super(craftedRecipe, quantity);
        }
    }
}
```

## Architecture & Concepts
CraftRecipeEvent is a message-oriented data object, not a service. It serves as a fundamental communication contract within the server's event-driven architecture, specifically for the crafting domain. This class and its nested subclasses, Pre and Post, are central to the server's ability to manage, validate, and react to item crafting operations in a decoupled and extensible manner.

The design embodies the **Observer** pattern and a **Vetoable State Change** model.
*   **CraftRecipeEvent.Pre:** This event is dispatched *before* a crafting operation is executed. It signals an *intent* to craft. Systems listening for this event act as validators or gatekeepers. They can inspect the proposed recipe and quantity and, by calling the inherited `setCancelled(true)` method, prevent the operation from proceeding. This is critical for implementing custom crafting rules, resource checks, or permission systems.
*   **CraftRecipeEvent.Post:** This event is dispatched *after* a crafting operation has successfully completed. It serves as an immutable notification of a world state change. Systems listening for this event can reliably trigger side effects, such as granting achievements, updating player statistics, or logging the action, knowing the craft has already occurred.

This two-phase event model is a cornerstone of the engine's transactional logic, ensuring that game state changes are predictable and can be intercepted by other systems.

### Lifecycle & Ownership
- **Creation:** An instance of CraftRecipeEvent (specifically Pre or Post) is created by the core crafting system when a player or entity initiates a craft. A `Pre` event is instantiated first to begin the validation phase. If and only if the `Pre` event is not cancelled by any listener, the crafting logic proceeds, and a `Post` event is instantiated and dispatched upon completion.
- **Scope:** The lifecycle of a CraftRecipeEvent instance is extremely brief. It exists only for the duration of its propagation through the event bus to all registered listeners. It is a fire-and-forget data container.
- **Destruction:** The object is managed entirely by the Java Garbage Collector. Once the event bus has finished dispatching it, all references are dropped, and it becomes eligible for collection. It holds no native resources and requires no manual cleanup.

## Internal State & Concurrency
- **State:** The core state of the event (the `craftedRecipe` and `quantity`) is **immutable**. These fields are final and are set only once during construction. This guarantees that all listeners receive the exact same, unaltered information about the crafting operation. The only mutable state is the `cancelled` flag inherited from CancellableEcsEvent, which is exclusively relevant to the `Pre` event subclass.
- **Thread Safety:** This event object is **not thread-safe** for modification but is safe for reading. It is designed to be created, dispatched, and processed within a single, well-defined thread, typically the main server game loop or "tick" thread. Listeners should not retain references to the event and attempt to access it from other threads. The immutability of its data payload mitigates most concurrency risks, but the event lifecycle itself is bound to the server's single-threaded event processing model.

## API Surface
The public contract is minimal, focusing on data access. The primary interaction for the `Pre` subclass involves the cancellation mechanism inherited from its parent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getCraftedRecipe() | CraftingRecipe | O(1) | Returns the non-null recipe that is the subject of the event. |
| getQuantity() | int | O(1) | Returns the number of items being crafted. |
| isCancelled() | boolean | O(1) | (Inherited) Checks if the `Pre` event has been cancelled by a listener. |
| setCancelled(boolean) | void | O(1) | (Inherited) Used by listeners of the `Pre` event to veto the crafting operation. |

## Integration Patterns

### Standard Usage
The correct pattern is to create a system that registers with the server's event bus and listens for the specific `Pre` or `Post` subclass.

```java
// Example of a system that prevents crafting certain recipes
public class RecipeGatekeeperSystem {

    @EventHandler
    public void onPreCraft(CraftRecipeEvent.Pre event) {
        // Prevent crafting of a "forbidden_item"
        if (event.getCraftedRecipe().getId().equals("forbidden_item")) {
            // Veto the operation. The core crafting logic will see this
            // and halt the process.
            event.setCancelled(true);
        }
    }

    @EventHandler
    public void onPostCraft(CraftRecipeEvent.Post event) {
        // This code only runs if the craft was successful (not cancelled).
        // Grant an achievement for crafting a "special_sword".
        if (event.getCraftedRecipe().getId().equals("special_sword")) {
            AchievementManager.grant(event.getPlayer(), "master_crafter");
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Listening to the Abstract Class:** Do not listen for the abstract `CraftRecipeEvent`. This is ambiguous and forces you to perform an `instanceof` check to determine if it is a `Pre` or `Post` event. Always subscribe directly to the concrete subclass you need to handle.
- **Assuming Completion on `Pre` Event:** Never trigger permanent side effects (like granting achievements or consuming rare items) when handling a `Pre` event. It provides no guarantee that the craft will actually complete, as a subsequent listener may cancel it.
- **Blocking Operations:** Event handlers execute on the main server thread. Performing file I/O, database queries, or other long-running tasks within a handler will freeze the entire server tick. Offload heavy work to an asynchronous task queue.
- **Manual Instantiation and Firing:** Application-level code should not create and fire these events. They are part of the core engine's internal logic. Doing so can desynchronize game state unless you are implementing a fully custom crafting system that replaces the default one.

## Data Pipeline
The flow of data and control for a crafting operation is orchestrated by the two-phase event dispatch.

> Flow:
> Player Crafting Request -> Core Crafting System -> **new CraftRecipeEvent.Pre()** -> Event Bus Dispatch -> Listeners (Validation, Permissions) -> Core Crafting System (checks `isCancelled()`) -> (If not cancelled) Item & Inventory State Change -> **new CraftRecipeEvent.Post()** -> Event Bus Dispatch -> Listeners (Achievements, Logging, Statistics)

