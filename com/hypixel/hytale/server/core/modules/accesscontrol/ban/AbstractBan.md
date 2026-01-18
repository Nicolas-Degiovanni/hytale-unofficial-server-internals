---
description: Architectural reference for AbstractBan
---

# AbstractBan

**Package:** com.hypixel.hytale.server.core.modules.accesscontrol.ban
**Type:** Base Class / Data Model

## Definition
```java
// Signature
abstract class AbstractBan implements Ban {
```

## Architecture & Concepts
AbstractBan is the foundational data model for all ban-related records within the server's access control module. It is not a service or a manager, but rather an immutable data contract that defines the essential properties of any ban action.

Its primary architectural role is to enforce a consistent structure for ban information: the **target** of the ban, the **issuer** of the ban, the **timestamp** of the event, and an optional **reason**. By centralizing this common state, it ensures that all concrete ban types, such as PermanentBan or TemporaryBan, are compatible with the core systems that persist, retrieve, and evaluate ban records.

This class functions as a Data Transfer Object (DTO) base, designed to carry information between different layers of the server architecture. It facilitates the flow of data from high-level moderation commands, through business logic services, and down to the persistence layer where it is typically serialized to a database.

## Lifecycle & Ownership
- **Creation:** As an abstract class, AbstractBan cannot be instantiated directly. Concrete subclasses are created by a factory or service, such as a BanManager, in response to a moderation event. For example, executing a ban command triggers the instantiation of a PermanentBan or TemporaryBan object with the relevant details.
- **Scope:** The lifetime of an AbstractBan instance is typically transactional and short-lived. An object is created, immediately serialized for persistent storage, and then becomes eligible for garbage collection. It is rehydrated from the database only when needed for inspection, modification (e.g., unbanning), or to enforce an access control check.
- **Destruction:** Ownership is managed by the Java Garbage Collector. Since instances are immutable and hold no external resources like file handles or network connections, no explicit cleanup is required.

## Internal State & Concurrency
- **State:** **Immutable**. All fields are declared final and are assigned only once within the constructor. This design guarantees that a ban record, once created, represents an unchanging historical fact. The use of Optional for the reason field ensures null-safety for this optional data point.
- **Thread Safety:** **Inherently thread-safe**. Due to its complete immutability, an instance of AbstractBan or any of its subclasses can be safely read and shared across multiple threads without any form of synchronization or locking. This is critical for high-performance access control systems where multiple player login requests may be processed concurrently.

## API Surface
The public API is composed entirely of non-mutating accessor methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getTarget() | UUID | O(1) | Retrieves the unique identifier of the banned entity. |
| getBy() | UUID | O(1) | Retrieves the unique identifier of the moderator or system that issued the ban. |
| getTimestamp() | Instant | O(1) | Returns the precise UTC timestamp when the ban was created. |
| getReason() | Optional<String> | O(1) | Returns the justification for the ban. The Optional is empty if no reason was provided. |
| toJsonObject() | JsonObject | O(1) | Serializes the complete state of the ban into a JsonObject for persistence or network transport. |

## Integration Patterns

### Standard Usage
Developers do not interact with AbstractBan directly but rather with its concrete implementations, which are typically provided by a repository or service. The primary interaction is reading data from an existing ban record.

```java
// A service retrieves a ban record, which is an implementation of AbstractBan
Ban currentBan = banRepository.findActiveBanForPlayer(playerUUID);

if (currentBan != null) {
    // The record can be safely inspected without locks
    log.warn("Login denied for " + currentBan.getTarget());
    currentBan.getReason().ifPresent(reason -> log.warn("Reason: " + reason));

    // Serialize for network transmission to a client or admin panel
    JsonObject banJson = currentBan.toJsonObject();
    networkChannel.send(new PlayerBanPacket(banJson));
}
```

### Anti-Patterns (Do NOT do this)
- **Replicating Logic:** Do not create custom classes that mimic the structure of AbstractBan. To introduce a new type of ban, you **must** extend AbstractBan to ensure it is correctly handled by the persistence and access control systems.
- **State Modification:** Do not attempt to modify the state of a ban object using reflection. Ban records are designed as immutable audit logs. To alter a ban (e.g., to lift it or change its reason), the correct pattern is to create a new, separate moderation event record that supersedes the original.

## Data Pipeline
AbstractBan serves as the data payload within the server's moderation and access control data pipeline. It represents a ban as it moves from command to persistent storage and back.

> Flow (Creation):
> Moderation Command -> BanService -> **(new ConcreteBan extends AbstractBan)** -> BanRepository -> Database (as JSON)

> Flow (Enforcement):
> Player Login Request -> AccessControlModule -> BanRepository -> **(Hydrated ConcreteBan instance)** -> Connection Rejected

