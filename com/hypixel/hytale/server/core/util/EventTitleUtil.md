---
description: Architectural reference for EventTitleUtil
---

# EventTitleUtil

**Package:** com.hypixel.hytale.server.core.util
**Type:** Utility

## Definition
```java
// Signature
public class EventTitleUtil {
```

## Architecture & Concepts
The EventTitleUtil class is a stateless, server-side utility that provides a high-level, declarative API for controlling the large, cinematic "event titles" displayed on a player's screen. It serves as a crucial facade, abstracting the underlying network protocol from core game logic systems like questing, zone transitions, or world events.

Its primary architectural function is to provide a single, authoritative entry point for a common UI operation, ensuring consistency in presentation and behavior. The class design offers multiple levels of granularity for dispatching titles:
*   **Universe-level:** Broadcast to every player on the server.
*   **World-level:** Target all players within a specific world instance.
*   **Player-level:** Target a single, specific player.

This tiered approach simplifies game logic, as developers can select the appropriate broadcast scope without needing to manually iterate through player or world collections.

## Lifecycle & Ownership
- **Creation:** As a static utility class, EventTitleUtil is never instantiated. Its methods are loaded by the JVM and become available for the entire application lifecycle.
- **Scope:** Application-wide. It persists as long as the server process is running.
- **Destruction:** There is no instance to destroy. The class is unloaded when the JVM shuts down.

## Internal State & Concurrency
- **State:** EventTitleUtil is **stateless**. It contains no mutable instance or static fields, only immutable constants. All operations are pure functions that transform input arguments into network packets.
- **Thread Safety:** The class itself is inherently thread-safe due to its stateless nature. However, callers must be aware of the threading context of the objects they pass into its methods.
    - The `showEventTitleToUniverse` method correctly demonstrates the required concurrency pattern: it iterates worlds and schedules the title display logic onto each world's respective thread using `world.execute`. This prevents race conditions when accessing a world's player list.
    - **WARNING:** Any direct calls to world-scoped methods like `showEventTitleToWorld` must be performed from that world's primary game loop thread or be explicitly scheduled onto it. Failure to do so will result in concurrency violations. Player-scoped methods are generally safe if the PlayerRef object is accessed from a thread-safe context.

## API Surface
The public API consists entirely of static methods for showing or hiding titles at different scopes.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| showEventTitleToUniverse(...) | static void | O(N) | Broadcasts a title to all players across all worlds. N is the total number of players on the server. |
| showEventTitleToWorld(...) | static void | O(P) | Broadcasts a title to all players in a specific world. P is the number of players in that world. |
| hideEventTitleFromWorld(...) | static void | O(P) | Hides the active title for all players in a specific world. |
| showEventTitleToPlayer(...) | static void | O(1) | Sends a title directly to a single player. Multiple overloads exist for convenience. |
| hideEventTitleFromPlayer(...) | static void | O(1) | Hides the active title for a single player. |

## Integration Patterns

### Standard Usage
Game systems should invoke the static methods directly, choosing the method that matches the desired operational scope. For example, a zone trigger system would target a specific player.

```java
// A player enters a new zone, show them a title.
PlayerRef player = ...; // Obtain PlayerRef from an event
Message title = Message.fromString("The Sunken City");
Message subtitle = Message.fromString("Beware the depths");

EventTitleUtil.showEventTitleToPlayer(player, title, subtitle, false);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The class has no public constructor and provides no instance-level functionality. Do not attempt to create an instance with `new EventTitleUtil()`.
- **Unsafe Threading:** Calling a world-scoped method from an asynchronous task or a different world's thread without proper scheduling is a severe anti-pattern that will lead to data corruption.

    ```java
    // DANGEROUS: Calling from the wrong thread
    World myWorld = ...;
    Store<EntityStore> store = myWorld.getEntityStore().getStore();

    // This call is only safe if the current thread is myWorld's main thread.
    EventTitleUtil.showEventTitleToWorld(title, subtitle, false, null, 4.0F, 1.5F, 1.5F, store);

    // CORRECT: Schedule the work on the world's thread
    myWorld.execute(() -> {
        EventTitleUtil.showEventTitleToWorld(title, subtitle, false, null, 4.0F, 1.5F, 1.5F, store);
    });
    ```

## Data Pipeline
EventTitleUtil acts as a translator between high-level server game logic and the low-level client-server packet protocol.

> Flow:
> Server Game Logic (e.g., Quest System) -> **EventTitleUtil.showEventTitleToPlayer()** -> new ShowEventTitle Packet -> PlayerRef PacketHandler -> Network Serialization -> Client UI System

