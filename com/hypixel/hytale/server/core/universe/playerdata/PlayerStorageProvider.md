---
description: Architectural reference for PlayerStorageProvider
---

# PlayerStorageProvider

**Package:** com.hypixel.hytale.server.core.universe.playerdata
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface PlayerStorageProvider {
```

## Architecture & Concepts
The PlayerStorageProvider interface defines a fundamental contract for abstracting the physical storage mechanism of player data on a server. It acts as a factory or provider in a Strategy design pattern, decoupling the core server logic from the specifics of how and where player information is persisted, such as in flat files or a database.

The most critical architectural feature is the static CODEC field. This utilizes a BuilderCodecMapCodec, a powerful serialization mechanism that allows the server to dynamically select and configure a concrete implementation of PlayerStorageProvider at runtime. This is typically driven by server or world configuration files. This pattern makes the player data backend highly extensible, enabling server administrators to plug in different storage systems (e.g., file-based for single-player, database-backed for large multiplayer servers) without altering core game code.

This interface is the entry point for the entire player data persistence layer. A higher-level service, such as a UniverseManager, will hold an instance of a concrete provider and use it to retrieve the PlayerStorage object responsible for I/O operations.

## Lifecycle & Ownership
As an interface, PlayerStorageProvider itself has no lifecycle. The following pertains to the concrete implementation class that is active at runtime.

- **Creation:** A concrete provider instance is deserialized and instantiated by the server's world-loading machinery during universe initialization. The specific implementation chosen is determined by configuration data, which is processed by the static CODEC.
- **Scope:** The provider instance is scoped to the lifecycle of the game universe or world it serves. It persists as long as the world is loaded and active.
- **Destruction:** The instance is eligible for garbage collection when the server shuts down or the associated world is unloaded. Implementations managing persistent connections (e.g., to a database) must handle their own resource cleanup.

## Internal State & Concurrency
- **State:** This interface is stateless by definition. However, concrete implementations are expected to be stateful. They will manage internal state such as file paths, database connection pools, or in-memory caches.
- **Thread Safety:** The interface contract does not enforce thread safety. Implementations **must be** thread-safe. Multiple server threads, each handling different player sessions, will concurrently request PlayerStorage instances. Implementations are responsible for ensuring that internal state is managed safely, typically through synchronization or by using thread-safe collections and connection managers.

## API Surface
The public contract is minimal, focusing solely on its factory role.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPlayerStorage() | PlayerStorage | Varies | Retrieves the primary PlayerStorage instance. Complexity depends on the implementation; may be O(1) for a cached singleton or involve a lookup. |

## Integration Patterns

### Standard Usage
The provider is typically retrieved from a central service or context during server initialization. It is then used to access the main storage system.

```java
// Within a high-level server management class
PlayerStorageProvider provider = worldConfig.getPlayerStorageProvider();
PlayerStorage storage = provider.getPlayerStorage();

// Now use the storage object to load/save player data
storage.savePlayerData(player);
```

### Anti-Patterns (Do NOT do this)
- **Implementation-Specific Code:** Do not cast the provider to a concrete type (e.g., FilePlayerStorageProvider). This violates the abstraction and defeats the purpose of the interface, making the system brittle and difficult to configure.
- **Ignoring the Provider:** Do not attempt to create PlayerStorage instances directly. The provider is the single source of truth and may perform critical initialization or connection management.

## Data Pipeline
The PlayerStorageProvider is a key component in the data flow for initializing the server's persistence layer.

> Flow:
> Server Configuration File (e.g., world.json) -> Server Bootstrap -> **PlayerStorageProvider (Deserialized via CODEC)** -> UniverseManager -> `getPlayerStorage()` call -> PlayerStorage Instance

