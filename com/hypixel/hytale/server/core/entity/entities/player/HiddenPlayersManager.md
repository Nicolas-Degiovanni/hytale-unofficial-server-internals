---
description: Architectural reference for HiddenPlayersManager
---

# HiddenPlayersManager

**Package:** com.hypixel.hytale.server.core.entity.entities.player
**Type:** Transient State Component

## Definition
```java
// Signature
public class HiddenPlayersManager {
```

## Architecture & Concepts
The HiddenPlayersManager is a server-side state management component responsible for tracking the visibility status of players. It serves as a centralized, authoritative source of truth for which players are intentionally hidden from others, typically for administrative, moderation, or gameplay purposes (e.g., a "vanish" mode).

This class does not implement the logic for hiding entities itself. Instead, it acts as a simple data store. Higher-level systems, such as the server's entity replication and network visibility subsystems, query this manager to make decisions. Before sending entity data for a given player to a client, the visibility system will consult this manager. If `isPlayerHidden` returns true, the entity packets for that player will be suppressed for other clients, effectively making them invisible.

Its design prioritizes thread safety and performance, making it suitable for high-concurrency server environments where multiple threads (game ticks, network I/O, command processors) may need to query or modify player visibility status simultaneously.

### Lifecycle & Ownership
- **Creation:** An instance of HiddenPlayersManager is typically created and owned by a higher-level container object, such as a `World` or `ServerZone` instance. It is not a global singleton.
- **Scope:** The lifecycle of a HiddenPlayersManager is strictly bound to its owner. It persists as long as the world or zone it manages is loaded and active. A separate instance exists for each distinct world context.
- **Destruction:** The object is eligible for garbage collection when its owning `World` or `ServerZone` is unloaded. It has no explicit cleanup or shutdown methods.

## Internal State & Concurrency
- **State:** The manager's state is mutable and consists of a single `Set` of player UUIDs. This set represents the collection of all players currently in a hidden state.
- **Thread Safety:** This class is **fully thread-safe**. The internal state is stored in a `Set` created from `ConcurrentHashMap.newKeySet()`. All public methods (`hidePlayer`, `showPlayer`, `isPlayerHidden`) are backed by the atomic, lock-free operations of the `ConcurrentHashMap`, guaranteeing safe concurrent access without the need for external synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| hidePlayer(UUID uuid) | void | O(1) avg | Atomically adds a player's UUID to the set of hidden players. |
| showPlayer(UUID uuid) | void | O(1) avg | Atomically removes a player's UUID from the set of hidden players. |
| isPlayerHidden(UUID uuid) | boolean | O(1) avg | Checks if a player's UUID is present in the set. Returns true if hidden. |

## Integration Patterns

### Standard Usage
The manager should be retrieved from its owning context (e.g., the `World` object). Game logic, such as command handlers or event listeners, modifies the state, while core engine systems query it.

```java
// Example: A command handler for a /vanish command
World world = player.getWorld();
HiddenPlayersManager visibilityManager = world.getHiddenPlayersManager();

// Toggle visibility state
if (visibilityManager.isPlayerHidden(player.getUUID())) {
    visibilityManager.showPlayer(player.getUUID());
    player.sendMessage("You are now visible.");
} else {
    visibilityManager.hidePlayer(player.getUUID());
    player.sendMessage("You are now hidden.");
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new HiddenPlayersManager()`. This would create a disconnected state store that the core server systems are unaware of. Always retrieve the context-specific instance from its owner (e.g., the `World`).
- **Assumed State:** Do not cache the result of `isPlayerHidden` for extended periods. The state can be changed at any time by another thread. Query the manager at the point where the visibility decision needs to be made.

## Data Pipeline
The HiddenPlayersManager acts as a state store, not a processing node. Data flows into it from control logic and is queried out by engine systems.

> **Input Flow (State Modification):**
> Admin Command -> Command Handler -> **HiddenPlayersManager.hidePlayer(uuid)** -> Internal State Updated

> **Query Flow (State-Based Decision):**
> Entity Replication System -> **HiddenPlayersManager.isPlayerHidden(uuid)** -> Boolean Decision -> Suppress Network Packet to Client

