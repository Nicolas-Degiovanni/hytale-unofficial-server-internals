---
description: Architectural reference for KillTrackerResource
---

# KillTrackerResource

**Package:** com.hypixel.hytale.builtin.adventure.npcobjectives.resources
**Type:** Transient State Component

## Definition
```java
// Signature
public class KillTrackerResource implements Resource<EntityStore> {
```

## Architecture & Concepts
The KillTrackerResource is a world-level data component responsible for maintaining a registry of all active kill-based objectives. As a Hytale *Resource*, it does not contain logic itself but rather serves as a state container that is attached to a broader entity, in this case, an EntityStore (representing a world).

Its primary function is to act as a central, queryable list for systems that process entity death events. When an entity is defeated, other systems can retrieve this resource from the world to efficiently check if the kill satisfies any ongoing quests or tasks. This decouples the quest definition systems from the core entity event processing loop, allowing for a clean separation of concerns.

The lifecycle of this resource is tied directly to the world it inhabits. It is instantiated when the world loads and is destroyed when the world unloads.

### Lifecycle & Ownership
-   **Creation:** An instance of KillTrackerResource is attached to a world's EntityStore by the server's resource management system. This is typically triggered by the initialization of the NPCObjectivesPlugin, which registers the corresponding ResourceType. It is not created manually by developers.
-   **Scope:** The resource exists for the entire lifetime of a server-side world instance. Its internal state persists as long as the world is loaded in memory.
-   **Destruction:** The resource is marked for garbage collection when its owning EntityStore is unloaded, for example, during a server shutdown or when a world is no longer active.

## Internal State & Concurrency
-   **State:** The core state is a single, mutable list of KillTaskTransaction objects. This list is dynamic, growing and shrinking as players accept and complete kill-related quests. The implementation uses a fastutil ObjectArrayList for performance.

-   **Thread Safety:** This class is **not thread-safe**. All interactions with its methods must be performed on the main server thread. The internal list is not protected by locks or concurrent collections. Unsynchronized access from worker threads (e.g., networking or physics threads) will result in state corruption or a ConcurrentModificationException.

    **WARNING:** External systems are responsible for ensuring all calls to this resource are synchronized with the main game loop.

## API Surface
The public API is minimal, focusing exclusively on managing the list of tracked tasks.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| watch(KillTaskTransaction task) | void | O(1) | Adds a new kill task to the tracking list. |
| unwatch(KillTaskTransaction task) | void | O(N) | Removes a kill task from the list. Requires a linear scan. |
| getKillTasks() | List | O(1) | Returns a direct reference to the internal list of tasks. |
| getResourceType() | ResourceType | O(1) | Static accessor to retrieve the registered type of this resource. |
| clone() | Resource | O(1) | Creates a new, empty KillTrackerResource. Does not copy tasks. |

## Integration Patterns

### Standard Usage
The intended pattern involves retrieving the resource from a world's EntityStore and using it to register or query tasks. This is typically done within a quest management system or an entity death event listener.

```java
// Example: An event listener processing an entity death
void onEntityDeath(EntityDeathEvent event) {
    // Retrieve the resource from the world's central storage
    EntityStore worldStore = event.getWorld().getEntityStore();
    KillTrackerResource tracker = worldStore.getResource(KillTrackerResource.getResourceType());

    if (tracker == null) {
        return; // No kill tasks are active in this world
    }

    // Iterate over a copy to prevent concurrent modification issues
    for (KillTaskTransaction task : new ObjectArrayList<>(tracker.getKillTasks())) {
        task.onEntityKilled(event.getEntity());
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new KillTrackerResource()`. The resource must be managed by the engine's resource system and retrieved from an EntityStore. Direct instantiation results in an unmanaged object that the game engine is unaware of.

-   **Modifying the Returned List:** The getKillTasks method returns a direct, mutable reference to the internal list for performance reasons. Modifying this list directly from outside the class can break the state machine of the quest system.
    ```java
    // BAD: This will corrupt the state for all quest systems
    tracker.getKillTasks().clear();
    ```

-   **Asynchronous Modification:** Do not call watch or unwatch from any thread other than the main server thread. This will lead to race conditions and unpredictable behavior.

## Data Pipeline
The KillTrackerResource serves as a simple data repository in a larger event-driven pipeline. It does not transform data but rather holds it for other systems to process.

> Flow:
> Quest System -> watch(task) -> **KillTrackerResource** (stores task) -> Entity Death Event -> Event Listener -> getKillTasks() -> Quest Progress System

