---
description: Architectural reference for NotificationUtil
---

# NotificationUtil

**Package:** com.hypixel.hytale.server.core.util
**Type:** Utility

## Definition
```java
// Signature
public class NotificationUtil {
```

## Architecture & Concepts
The NotificationUtil class is a static utility that provides a high-level, simplified API for sending user-facing notifications. It serves as a facade over the low-level networking layer, abstracting the construction and dispatch of Notification packets.

Its primary architectural role is to act as a centralized and thread-safe entry point for any server system that needs to communicate information directly to a player's user interface. The utility offers methods with varying scopes:

*   **Universe-level:** Broadcasts a notification to every player across all active worlds.
*   **World-level:** Broadcasts a notification to all players within a specific world instance.
*   **Player-level:** Sends a notification to a single, targeted player via their PacketHandler.

This tiered approach allows game logic to operate at the appropriate level of abstraction without needing direct knowledge of the underlying player connection management or world-threading models.

## Lifecycle & Ownership
- **Creation:** As a static utility class, NotificationUtil is never instantiated. Its bytecode is loaded by the JVM ClassLoader upon first reference, and it exists as a single, static definition.
- **Scope:** Application-level. The class and its static methods are available for the entire lifetime of the server process.
- **Destruction:** The class is unloaded from memory when the JVM shuts down. There is no instance-specific cleanup or destruction logic.

## Internal State & Concurrency
- **State:** NotificationUtil is entirely **stateless**. It contains no member fields and all data it operates on is provided as method arguments or retrieved from globally accessible singletons like Universe.

- **Thread Safety:** This class is **thread-safe** by design. The universe-scoped and world-scoped methods are safe to invoke from any thread. The `sendNotificationToUniverse` method achieves this by iterating through each World and scheduling the notification dispatch onto that specific world's dedicated thread via the `world.execute` method. This pattern ensures that iteration over a world's player list and access to their components occurs without concurrency conflicts.

    **Warning:** While the universe and world methods manage their own threading, the player-specific `sendNotification(PacketHandler, ...)` method performs no such dispatching. It is the caller's responsibility to ensure this method is invoked from the correct network or game-loop thread associated with the target PacketHandler.

## API Surface
The public API consists of numerous static convenience overloads. The core methods from which others are derived are listed below.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| sendNotificationToUniverse(args...) | void | O(W * P) | Broadcasts a notification to all players (P) in all worlds (W). Dispatches work to each world's thread. |
| sendNotificationToWorld(args...) | void | O(P) | Broadcasts a notification to all players (P) in a given world. Assumes execution on the correct world thread. |
| sendNotification(handler, args...) | void | O(1) | Constructs and sends a single Notification packet to a specific client via their PacketHandler. |

## Integration Patterns

### Standard Usage
The most common use case is broadcasting a server-wide message, such as for a global event or system announcement. This should be done from high-level game logic.

```java
// Announce a server-wide event to all players
import com.hypixel.hytale.server.core.util.NotificationUtil;
import com.hypixel.hytale.protocol.packets.interface_.NotificationStyle;

// ... inside some game logic method
NotificationUtil.sendNotificationToUniverse(
    "A server-wide event is starting in 5 minutes!",
    NotificationStyle.Warning
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The class has no public constructor and contains only static methods. Do not attempt to create an instance with `new NotificationUtil()`.
- **High-Frequency Broadcasting:** Do not call `sendNotificationToUniverse` on a frequent, per-tick basis. This will generate significant network traffic and spam players. These methods are intended for infrequent, important announcements.
- **Incorrect Threading for Direct Send:** Do not call the player-specific `sendNotification(PacketHandler, ...)` from an arbitrary worker thread. This can lead to race conditions in the networking layer. This method should only be called from a context that already owns the player's PacketHandler, such as within a world's tick loop.

## Data Pipeline
The data flow for a universe-wide broadcast demonstrates how the utility orchestrates cross-thread communication to ensure safety and correctness.

> Flow:
> Game Logic Call -> **NotificationUtil.sendNotificationToUniverse** -> Universe Singleton -> World Instance -> World Thread Scheduler (`world.execute`) -> PlayerRef Iteration -> PacketHandler -> Network Layer (Packet Serialization) -> Client UI

