---
description: Architectural reference for PlayerCraftEvent
---

# PlayerCraftEvent

**Package:** com.hypixel.hytale.server.core.event.events.player
**Type:** Transient Data Model

## Definition
```java
// Signature
@Deprecated(forRemoval = true)
public class PlayerCraftEvent extends PlayerEvent<String> {
```

## Architecture & Concepts
The PlayerCraftEvent is an immutable data carrier object that represents a single, discrete crafting action performed by a Player on the server. Within the server's event-driven architecture, its primary role is to decouple the core crafting logic from subsequent systems that need to react to a successful craft, such as inventory management, achievement tracking, or server-side logging.

**CRITICAL WARNING: This event is deprecated and scheduled for removal.** Its use indicates reliance on an obsolete part of the crafting system. All new development must use the modern equivalent, and existing systems should be migrated away from listening for this event. Its continued presence is for backward compatibility during a transition period only.

This class follows the Command Query Responsibility Segregation (CQRS) pattern, where the event itself is a pure data record of something that has already occurred. It carries no logic and only serves to inform other systems of a state change.

## Lifecycle & Ownership
- **Creation:** An instance of PlayerCraftEvent is created exclusively by the server's internal crafting validation system immediately after a player's crafting request is successfully processed and the resulting items are ready to be awarded.
- **Scope:** The object's lifetime is extremely short and is scoped to a single event dispatch cycle. It is created, posted to the server's central Event Bus, and becomes eligible for garbage collection once all registered listeners have processed it.
- **Destruction:** Managed entirely by the Java Garbage Collector. There is no manual cleanup or resource release associated with this object.

## Internal State & Concurrency
- **State:** Immutable. The craftedRecipe and quantity fields are final and are set only once during construction. This guarantees that all listeners receive the exact same, unaltered data, preventing race conditions and unpredictable side effects between handlers.
- **Thread Safety:** Inherently thread-safe. Due to its immutability, an instance of PlayerCraftEvent can be safely passed between threads without any need for locks, synchronization, or other concurrency controls. This is essential for the event bus, which may dispatch events to listeners operating on different threads.

## API Surface
The public API is minimal, providing read-only access to the event's payload.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getCraftedRecipe() | CraftingRecipe | O(1) | Returns the recipe definition for the item that was crafted. |
| getQuantity() | int | O(1) | Returns the number of items produced by the crafting operation. |

## Integration Patterns

### Standard Usage
This pattern is for historical context only. **Do not write new code that uses this event.** A typical listener would be registered with the server's event bus and would implement a handler method that is invoked when a PlayerCraftEvent is posted.

```java
// Example of a legacy achievement listener
@Subscribe
public void onPlayerCraft(PlayerCraftEvent event) {
    Player player = event.getPlayer();
    CraftingRecipe recipe = event.getCraftedRecipe();

    // Check if this craft unlocks an achievement
    if (recipe.getId().equals("special_sword")) {
        player.getAchievementService().unlock("achievement.first_sword");
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Listening for this Event:** The primary anti-pattern is adding any new listeners for this event. All new logic should interface with the replacement crafting event system.
- **Direct Instantiation:** Event listeners or other services must never create an instance of PlayerCraftEvent using its constructor. These events are authoritative records of actions that have already happened and are only ever created by the core crafting system.
- **Ignoring Deprecation:** Continuing to rely on this event in existing code is a technical debt risk, as it will be removed in a future version, causing a hard failure.

## Data Pipeline
The flow for this event originates from a player action and terminates after being broadcast to all interested systems.

> Flow:
> Player Crafting Request -> Server Crafting System Validation -> **PlayerCraftEvent Instantiation** -> Server Event Bus -> Event Listeners (Inventory, Achievements, Logging)<ctrl63>

