---
description: Architectural reference for TimedBan
---

# TimedBan

**Package:** com.hypixel.hytale.server.core.modules.accesscontrol.ban
**Type:** Value Object

## Definition
```java
// Signature
public class TimedBan extends AbstractBan {
```

## Architecture & Concepts
The TimedBan class is an immutable data model representing a player sanction with a specific expiration date. It is a concrete implementation within the server's Access Control module, designed to encapsulate all state related to a temporary ban.

As a subclass of AbstractBan, it inherits the core identity of a ban (target, issuer, timestamp, reason) and adds the critical concept of a finite duration. Its primary role is to serve as a self-contained, serializable record that can be persisted to storage (e.g., a JSON file) and later rehydrated into a live object for enforcement checks. The class is designed for data integrity, ensuring that once a TimedBan is created, its properties cannot be altered.

This object is a fundamental building block for moderation systems, providing the logic to determine if a ban is currently active and to generate the user-facing disconnection message.

## Lifecycle & Ownership
- **Creation:** A TimedBan instance is created under two distinct circumstances:
    1. **Programmatic Instantiation:** When a server administrator or automated system issues a new temporary ban, the primary constructor is called with all required parameters (target UUID, issuer UUID, expiration, etc.). This is typically handled by a command processor or moderation service.
    2. **Deserialization:** During server startup, the ban management system reads persisted ban records from a data store. The static factory method, fromJsonObject, is invoked to reconstruct TimedBan objects from their JSON representation.

- **Scope:** The lifetime of a TimedBan object is managed by the service that holds it, usually a central BanManager or a similar registry. It persists in memory as long as the ban is considered relevant by the server, which is typically until after it expires and is purged.

- **Destruction:** Instances are eligible for garbage collection when they are no longer referenced. This occurs when a ban is explicitly removed (e.g., pardoned) or when an expired ban is purged from the active ban list by a cleanup task.

## Internal State & Concurrency
- **State:** The TimedBan object is **immutable**. All its fields, including the inherited ones from AbstractBan and its own expiresOn field, are set only once during construction. This design guarantees that the state of a ban record cannot change during its lifetime, which is critical for predictable and consistent enforcement.

- **Thread Safety:** This class is inherently **thread-safe**. Due to its immutability, instances can be safely shared and read by multiple threads without the need for locks or other synchronization primitives. This is essential for a high-performance server environment where connection handling and ban checks may occur on different worker threads.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fromJsonObject(JsonObject) | TimedBan | O(1) | **Static Factory.** Deserializes a JsonObject into a TimedBan instance. Throws JsonParseException on malformed input. |
| isInEffect() | boolean | O(1) | Checks if the ban's expiration time is after the current system time. This is the primary method for enforcement. |
| getType() | String | O(1) | Returns the unique type identifier "timed". Used for serialization and type discrimination. |
| getExpiresOn() | Instant | O(1) | Returns the exact moment in time when the ban will expire. |
| getDisconnectReason(UUID) | CompletableFuture | O(1) | Generates the formatted, human-readable message to be sent to a banned player upon a connection attempt. |
| toJsonObject() | JsonObject | O(1) | Serializes the TimedBan instance into its JsonObject representation for persistence. |

## Integration Patterns

### Standard Usage
A TimedBan object should not be used in isolation. It is intended to be managed by a higher-level service, such as a BanManager, which holds a collection of all active bans. The manager is responsible for retrieving the correct ban object for a connecting player and using it to enforce the sanction.

```java
// Example: Within a hypothetical BanManager service
public CompletableFuture<Optional<String>> getDisconnectReasonForPlayer(UUID playerUUID) {
    // Retrieve the ban from an in-memory map or database
    AbstractBan ban = this.activeBans.get(playerUUID);

    // Check if the ban is a TimedBan and is still active
    if (ban instanceof TimedBan && ban.isInEffect()) {
        return ban.getDisconnectReason(playerUUID);
    }

    return CompletableFuture.completedFuture(Optional.empty());
}
```

### Anti-Patterns (Do NOT do this)
- **State Modification:** Do not attempt to modify the state of a TimedBan object after creation using reflection. The system relies on its immutability for thread safety and data consistency.
- **Clock Manipulation:** The isInEffect logic depends on Instant.now(). Server logic should not be written in a way that is sensitive to minor system clock adjustments. Relying on this for critical, time-sensitive logic outside of ban enforcement is discouraged.
- **Incorrect Deserialization:** Do not manually parse the JSON. Always use the provided fromJsonObject factory method to ensure all fields are correctly validated and instantiated.

## Data Pipeline
The TimedBan class is a critical component in the data flow for both persisting and enforcing bans.

> **Persistence Flow (Serialization):**
> Ban Command -> **new TimedBan(...)** -> BanManager -> **TimedBan.toJsonObject()** -> JSON Persistence Layer -> Disk/Database

> **Enforcement Flow (Deserialization & Use):**
> Server Startup -> JSON Persistence Layer -> **TimedBan.fromJsonObject()** -> BanManager Cache
> ---
> Player Connection Event -> BanManager Cache -> **TimedBan.isInEffect()** -> **TimedBan.getDisconnectReason()** -> Disconnect Packet to Client

