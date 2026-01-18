---
description: Architectural reference for BanParser
---

# BanParser

**Package:** com.hypixel.hytale.server.core.modules.accesscontrol.ban
**Type:** Strategy Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface BanParser {
   Ban parse(JsonObject var1) throws JsonParseException;
}
```

## Architecture & Concepts
The BanParser interface defines a behavioral contract for deserializing a JSON representation into a concrete Ban object. As a functional interface, it serves as a key component of the Strategy Pattern within the server's access control module.

Its primary architectural purpose is to decouple the core ban management logic from the specific data format of a ban entry. This allows the system to evolve its ban data structure or support multiple legacy formats without altering the services that consume Ban objects, such as the BanManager or connection authenticators. Implementations of this interface encapsulate the logic for interpreting a specific version or structure of a ban's JSON data.

This component sits at the boundary between raw data persistence (e.g., a `bans.json` file) and the in-memory representation of server access control rules.

### Lifecycle & Ownership
- **Creation:** Implementations of BanParser are expected to be instantiated by a factory or dependency injection framework responsible for configuring the access control module. The specific implementation chosen may depend on server configuration or the version of the ban data being read.
- **Scope:** A BanParser implementation should be stateless and is therefore safe to be scoped as a singleton for the entire server session. Its lifecycle is tied to the service that uses it, typically the BanManager.
- **Destruction:** The object is subject to standard Java Garbage Collection when the owning service is destroyed, typically during server shutdown.

## Internal State & Concurrency
- **State:** The interface itself is stateless. All conforming implementations **must** be stateless. The `parse` method's logic should depend only on its input arguments. Storing state between calls is a severe anti-pattern that would violate the contract and introduce unpredictable behavior.
- **Thread Safety:** The contract is inherently thread-safe, provided that implementations are stateless. The `parse` method is a pure function that transforms a JsonObject into a Ban object without side effects. It can be safely invoked from multiple threads concurrently, for instance, during a parallelized reload of a large ban list.

## API Surface
The public contract consists of a single abstract method, as mandated by the FunctionalInterface annotation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| parse(JsonObject) | Ban | O(N) | Deserializes the provided JsonObject into a Ban domain object. Throws JsonParseException if the input is malformed or missing required fields. N is the number of tokens in the input object. |

## Integration Patterns

### Standard Usage
Implementations are typically provided to a higher-level service that manages the loading and querying of ban data. The parser is used as a strategy for data transformation.

```java
// An implementation of the parser is supplied to a manager
BanParser parser = new StandardBanParser(); // Example implementation
BanManager manager = new BanManager(banDataSource, parser);

// The manager uses the parser internally to load bans
manager.reloadBans();
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not create implementations that store instance variables modified during the `parse` call. This breaks thread safety and reusability.
- **Swallowing Exceptions:** Do not implement the `parse` method with a top-level try-catch block that returns null. The `throws JsonParseException` clause is a critical part of the contract. The calling service must be notified of data corruption to prevent silently ignoring invalid ban entries.

## Data Pipeline
This interface acts as a transformation step in the data flow from persistent storage to the in-memory access control system.

> Flow:
> Raw JSON String (from file/database) -> GSON Library -> JsonObject -> **BanParser Implementation** -> Ban Object -> BanManager Cache

