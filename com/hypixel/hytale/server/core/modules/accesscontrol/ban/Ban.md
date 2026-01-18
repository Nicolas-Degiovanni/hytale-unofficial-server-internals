---
description: Architectural reference for the Ban interface, a core contract within the server access control system.
---

# Ban

**Package:** com.hypixel.hytale.server.core.modules.accesscontrol.ban
**Type:** Contract / Data Transfer Object (DTO)

## Definition
```java
// Signature
public interface Ban extends AccessProvider {
```

## Architecture & Concepts
The Ban interface is a foundational contract within the server's access control module. It does not represent a service or manager, but rather an immutable data record defining a single ban action against a player. Its primary role is to serve as a standardized data structure for ban information retrieved from persistent storage.

A critical architectural aspect is its extension of the AccessProvider interface. This is a powerful application of the Strategy pattern, allowing a Ban object itself to be treated as a source of access control decisions. Instead of a service checking a Ban object's properties, the Ban object can be passed directly to a system that consumes AccessProviders, which then determines if access should be granted or denied. This decouples the access control logic from the specifics of *why* access is denied.

## Lifecycle & Ownership
As an interface, Ban does not have a concrete lifecycle. The following describes the lifecycle of the *data objects* that implement this contract.

- **Creation:** Concrete Ban objects are instantiated by a dedicated service, such as a BanService or BanRepository, typically in response to a moderation event (e.g., an in-game command or an external API call). They are created by deserializing data from a primary data store like a database.
- **Scope:** In-memory Ban objects are short-lived and scoped to a specific operation, most commonly a player's connection attempt. A Ban object is fetched, used for the access check, and then becomes eligible for garbage collection. The underlying ban *record* in the database is what persists across sessions.
- **Destruction:** The in-memory object is reclaimed by the garbage collector once it's out of scope. The persistent record is logically destroyed by an "unban" operation, which would delete or invalidate the entry in the database.

## Internal State & Concurrency
- **State:** The contract defined by the Ban interface is fundamentally **immutable**. All methods are accessors (getters) with no corresponding mutators (setters). Implementations are expected to be read-only snapshots of a ban at a specific point in time.
- **Thread Safety:** Due to its immutable nature, any correct implementation of the Ban interface is inherently **thread-safe**. A single Ban instance can be safely read and passed between multiple threads without locks or synchronization, which is critical for high-throughput connection handling.

## API Surface
The public contract consists entirely of non-blocking data accessors.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getTarget() | UUID | O(1) | Returns the unique identifier of the banned player. |
| getBy() | UUID | O(1) | Returns the unique identifier of the entity that issued the ban. |
| getTimestamp() | Instant | O(1) | Returns the exact moment the ban was created. |
| isInEffect() | boolean | O(1) | Determines if the ban is currently active. Essential for temporary bans. |
| getReason() | Optional<String> | O(1) | Provides the moderator-specified reason for the ban, if one exists. |
| getType() | String | O(1) | Returns a string identifier for the ban type (e.g., "PERMANENT", "TEMPORARY"). |
| toJsonObject() | JsonObject | O(N) | Serializes the ban's state into a JSON object for storage or network transfer. |

## Integration Patterns

### Standard Usage
The Ban interface is primarily consumed by higher-level access control systems during the player login sequence. The system retrieves a list of potential bans for a connecting player and evaluates them.

```java
// A service retrieves a Ban object for a connecting player
Optional<Ban> potentialBan = banRepository.findActiveBanForPlayer(playerUuid);

// The Ban object is used to make an access control decision
if (potentialBan.isPresent() && potentialBan.get().isInEffect()) {
    connection.deny(potentialBan.get().getReason().orElse("You are banned."));
}
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not attempt to cast a Ban object to a concrete type to modify its state. A ban is a historical record; to change it, a new ban should be issued or the existing one should be formally revoked through the appropriate service.
- **Concrete Implementation Logic:** Systems should not depend on a specific implementation of Ban (e.g., `if (ban instanceof PermanentBan)`). All logic should be based on the methods provided by the interface, such as getType() or isInEffect().

## Data Pipeline
The Ban object is a key payload in the player connection and moderation data flows.

> **Player Connection Flow:**
> Player Connection Attempt -> ConnectionHandler -> BanRepository.findByTarget(uuid) -> **Ban** (object) -> AccessControlService -> Connection Denied Response

> **Moderation Flow:**
> Moderator Command -> CommandParser -> BanService.createBan(...) -> **Ban** (object) -> BanRepository.save(ban) -> Database Record

