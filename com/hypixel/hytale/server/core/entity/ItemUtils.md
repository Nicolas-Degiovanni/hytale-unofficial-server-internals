---
description: Architectural reference for ItemUtils
---

# ItemUtils

**Package:** com.hypixel.hytale.server.core.entity
**Type:** Utility

## Definition
```java
// Signature
public class ItemUtils {
```

## Architecture & Concepts
The **ItemUtils** class is a stateless, static utility that serves as a high-level orchestrator for common item-related world interactions. It encapsulates the complex, multi-step processes of an entity picking up, throwing, or dropping an item. This class acts as a critical bridge between gameplay logic and the low-level Entity Component System (ECS) state management.

Its primary architectural role is to provide a transactional and event-driven workflow for item manipulation. Instead of directly manipulating inventory and transform components, other systems invoke **ItemUtils** to ensure that all necessary side effects are correctly handled. This includes:

-   **Event Dispatch:** Firing cancellable events like **DropItemEvent** and **InteractivelyPickupItemEvent**, allowing other game modules to intercept and modify item interactions.
-   **Inventory Logic:** Correctly interacting with an entity's inventory containers, processing **ItemStackTransaction** objects, and handling remainder stacks.
-   **Entity Creation:** Spawning new item entities into the world via the **ComponentAccessor** when an item is thrown or dropped.
-   **Physics Calculation:** Determining initial velocity and position for thrown items based on the source entity's transform and head rotation.

By centralizing this logic, **ItemUtils** promotes consistency and reduces boilerplate code throughout the server codebase.

## Lifecycle & Ownership
-   **Creation:** As a class containing only static methods, **ItemUtils** is never instantiated. It is loaded by the Java ClassLoader at runtime.
-   **Scope:** The class and its methods are available for the entire lifetime of the server application. It has no instance-specific state or scope.
-   **Destruction:** Not applicable. The class is unloaded when the application terminates.

## Internal State & Concurrency
-   **State:** **ItemUtils** is fundamentally stateless. It contains no member fields and all data required for its operations is passed in as method arguments. Each method call is an independent, atomic transaction from the perspective of the class itself.

-   **Thread Safety:** The class itself is inherently thread-safe due to its stateless nature. However, the operations it performs are **NOT** thread-safe. All methods extensively use the **ComponentAccessor** to read from and write to the shared ECS state.

    **WARNING:** Invoking any method in **ItemUtils** from a thread other than the main server tick thread will lead to severe concurrency issues, data corruption, and server instability. All calls must be synchronized with the primary game loop.

## API Surface
All methods are static and operate on entities via a **Ref** and a **ComponentAccessor**.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| interactivelyPickupItem(ref, itemStack, origin, accessor) | void | Complex | Orchestrates an entity picking up an item. Fires an event, handles inventory transactions, and drops any remainder. |
| throwItem(ref, itemStack, throwSpeed, accessor) | Ref<EntityStore> | Complex | Throws an item from an entity's perspective. Calculates direction from head rotation and spawns a new item entity. Returns a reference to the new entity. |
| throwItem(ref, accessor, itemStack, direction, speed) | Ref<EntityStore> | Complex | Low-level method to throw an item from a specific position in a given direction. Spawns the new item entity. |
| dropItem(ref, itemStack, accessor) | Ref<EntityStore> | Complex | A convenience wrapper for **throwItem** with a default throw speed. Returns a reference to the new entity. |

*Complexity is marked as "Complex" because operations involve event dispatching, component lookups, and entity creation, which are not constant-time operations.*

## Integration Patterns

### Standard Usage
**ItemUtils** should be used by any system that needs to make an entity interact with an item in the world. The caller must have access to the entity's **Ref** and the current **ComponentAccessor**.

```java
// Example: A player entity dropping the currently held item.
// This logic would typically reside in a player action handler system.

ComponentAccessor<EntityStore> accessor = ...;
Ref<EntityStore> playerRef = ...;
Player playerComponent = accessor.getComponent(playerRef, Player.getComponentType());

// Assume we get the item to drop from the player's inventory
ItemStack itemToDrop = playerComponent.getInventory().takeFromHotbar(1);

if (itemToDrop != null && !itemToDrop.isEmpty()) {
    // Use ItemUtils to handle the entire drop process
    ItemUtils.dropItem(playerRef, itemToDrop, accessor);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** The class has no public constructor and provides no instance methods. Do not attempt to create an instance with `new ItemUtils()`.
-   **Asynchronous Invocation:** Never call **ItemUtils** methods from a separate thread (e.g., network thread, async task). This will bypass the server's tick-based synchronization and corrupt ECS state. All calls must originate from the main server thread.
-   **Ignoring Event System:** Do not attempt to replicate the logic within **ItemUtils** to bypass the event system. The events it fires are critical for other game modules to function correctly.

## Data Pipeline
The data flow through **ItemUtils** is transactional and event-driven. It transforms a high-level intent into a series of low-level ECS state changes.

**Example Flow for `throwItem`:**

> Player Input -> Action Handler -> **ItemUtils.throwItem(ref, item, speed, accessor)** -> Fires **DropItemEvent** -> Event Bus -> Listeners (may modify event) -> **ItemUtils** checks if cancelled -> Reads **HeadRotation** & **TransformComponent** from ECS -> Calculates throw vector -> Calls internal **ItemUtils.throwItem(ref, accessor, item, direction, speed)** -> Calls **ItemComponent.generateItemDrop** -> Creates new Entity **Holder** -> Calls **accessor.addEntity** -> New item entity exists in world -> Returns **Ref** of new entity.

