---
description: Architectural reference for InfiniteBan
---

# InfiniteBan

**Package:** com.hypixel.hytale.server.core.modules.accesscontrol.ban
**Type:** Value Object

## Definition
```java
// Signature
public class InfiniteBan extends AbstractBan {
```

## Architecture & Concepts
The InfiniteBan class is a concrete implementation of the AbstractBan contract. It represents a permanent, non-expiring sanction against a player identity (UUID). This class is a key component of the server's Access Control Module, providing a specific strategy for handling permanent bans.

Its primary architectural role is to encapsulate the state and behavior of a permanent ban, distinguishing it from other sanction types like TemporaryBan. The system relies on polymorphism; a managing service, such as a BanManager, interacts with the AbstractBan interface, and the specific runtime type (e.g., InfiniteBan) determines the outcome of checks like *isInEffect*.

This design decouples the core access control logic from the specifics of any single ban type, allowing new types of sanctions to be added without modifying the primary login and validation flows. The static factory method, fromJsonObject, indicates that these objects are primarily intended to be deserialized from a persistent data store, such as a JSON file or database.

## Lifecycle & Ownership
- **Creation:** InfiniteBan instances are not typically created directly. They are instantiated by a higher-level management service (e.g., BanManager) during two main scenarios:
    1.  **Deserialization:** When the server loads its ban list from a persistent source, the static fromJsonObject factory is called to reconstruct the object from its JSON representation.
    2.  **New Sanction:** When a server administrator issues a new permanent ban via a command, the managing service creates a new InfiniteBan instance before persisting it.

- **Scope:** The object instance is short-lived and transient. It exists in memory only for the duration of a specific operation, such as validating a player's connection attempt or responding to an API query. The *data* it represents is persistent, but the *object* is not.

- **Destruction:** Instances are eligible for garbage collection as soon as the operation that required them is complete and all local references are out of scope. They are not managed or pooled.

## Internal State & Concurrency
- **State:** **Immutable**. An InfiniteBan object's state, including the target UUID, issuing authority, timestamp, and reason, is established at construction and cannot be modified thereafter. This immutability is critical for predictable and safe behavior in a multi-threaded server environment.

- **Thread Safety:** **Fully thread-safe**. As an immutable value object, an InfiniteBan instance can be safely shared, passed between, and read by multiple threads without any need for external synchronization or locks.

## API Surface
The public API is minimal, adhering to the contract defined by AbstractBan. It focuses on state interrogation rather than mutation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fromJsonObject(JsonObject) | InfiniteBan | O(1) | **Static Factory.** Deserializes a JSON object into an InfiniteBan instance. Throws JsonParseException on malformed input. |
| isInEffect() | boolean | O(1) | Determines if the ban is active. For InfiniteBan, this method is hardcoded to always return true. |
| getType() | String | O(1) | Returns the unique type identifier "infinite". Used for serialization and type discrimination without reflection. |
| getDisconnectReason(UUID) | CompletableFuture | O(1) | Constructs and returns the disconnect message for a banned player. The operation is non-blocking and completes immediately. |

## Integration Patterns

### Standard Usage
Developers should never interact with this class directly. Interaction is mediated through a central access control or ban management service. The service handles loading, caching, and querying ban records.

```java
// Example: Within a hypothetical BanManager service
AccessControlModule service = serverContext.getService(AccessControlModule.class);
UUID connectingPlayerId = ...;

// The service returns an Optional<AbstractBan>
Optional<AbstractBan> banRecord = service.getBanRecord(connectingPlayerId);

if (banRecord.isPresent() && banRecord.get().isInEffect()) {
    // The polymorphic call to isInEffect() will return true for an InfiniteBan
    CompletableFuture<Optional<String>> reasonFuture = banRecord.get().getDisconnectReason(connectingPlayerId);
    
    // Disconnect the player using the provided reason
    reasonFuture.thenAccept(reasonOpt -> {
        playerConnection.disconnect(reasonOpt.orElse("You are banned."));
    });
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new InfiniteBan(...)`. Creating an instance of this class does not persist the ban. All sanctions must be created through the appropriate management service to ensure they are correctly recorded in the persistent data store.

- **Type Checking:** Avoid checking the specific type of a ban object. The system is designed to be polymorphic. Rely on the methods provided by the AbstractBan interface.
    ```java
    // BAD: Violates polymorphism
    if (ban instanceof InfiniteBan) {
        // ... custom logic
    }

    // GOOD: Relies on the object's contract
    if (ban.isInEffect()) {
        // ... generic logic
    }
    ```

## Data Pipeline
The InfiniteBan object primarily functions as a deserialized data record used during the player connection pipeline.

> Flow:
> Player Connection Event -> AccessControlModule -> Ban Data Store (e.g., bans.json) -> **InfiniteBan.fromJsonObject** -> In-Memory Ban Record -> **InfiniteBan.isInEffect()** Check -> Connection Rejected with Reason

