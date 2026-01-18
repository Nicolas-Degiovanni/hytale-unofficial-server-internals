---
description: Architectural reference for DataStoreProvider
---

# DataStoreProvider

**Package:** com.hypixel.hytale.server.core.universe.datastore
**Type:** Factory Interface

## Definition
```java
// Signature
public interface DataStoreProvider {
   BuilderCodecMapCodec<DataStoreProvider> CODEC = new BuilderCodecMapCodec<>("Type");

   <T> DataStore<T> create(BuilderCodec<T> var1);
}
```

## Architecture & Concepts
The DataStoreProvider interface defines a contract for an Abstract Factory. Its primary role is to decouple game logic from the underlying data persistence mechanism. It acts as a centralized factory for creating DataStore instances, which are specialized containers for persisting specific types of game data, such as player inventories or world state.

This abstraction allows the server to be configured with different storage backends—for example, an in-memory provider for testing, a file-based provider for single-player or small servers, or a database-backed provider for large-scale production environments—without altering the core game services that consume the data.

The static CODEC field is a critical component for configuration-driven instantiation. The server's configuration system uses this codec to deserialize a string identifier (e.g., "file" or "database") into a concrete DataStoreProvider implementation at startup. This makes the entire persistence layer pluggable.

## Lifecycle & Ownership
As an interface, DataStoreProvider itself has no lifecycle. The following describes the lifecycle of a concrete implementation.

-   **Creation:** A single, specific implementation of DataStoreProvider is instantiated by the server's core bootstrap sequence. The choice of implementation is read from a server configuration file and resolved using the static CODEC.
-   **Scope:** The provider instance is a long-lived object, typically scoped to the entire server session or a specific game "universe". It persists as long as the server is running.
-   **Destruction:** The provider is destroyed during the server shutdown sequence. Any underlying resources, such as database connections or file handles managed by the implementation, must be released at this time.

## Internal State & Concurrency
-   **State:** The interface is stateless. However, concrete implementations are expected to be stateful. For example, a file-based provider would maintain a root directory path, while a database provider would manage a connection pool.
-   **Thread Safety:** Implementations of this interface **must be thread-safe**. The create method may be invoked concurrently by different systems managing various game objects. All internal state within an implementation, such as connection pools or file caches, must be protected against race conditions.

## API Surface
The public contract is minimal, focusing exclusively on codec-based resolution and instance creation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | static BuilderCodecMapCodec | O(1) | A static codec used to map a configuration string to a concrete provider implementation. This is the entry point for deserializing the provider from server settings. |
| create(BuilderCodec var1) | DataStore | O(1) to O(N) | Factory method. Creates and returns a type-specific DataStore. Complexity depends on the implementation; it may involve I/O operations like opening files or acquiring database connections. |

## Integration Patterns

### Standard Usage
The provider is typically retrieved from a central service registry or dependency injection container. Game systems then use it to request DataStore instances for the specific data types they manage.

```java
// Retrieve the configured provider from a Universe or Service context
DataStoreProvider provider = universe.getDataStoreProvider();

// Request a DataStore specifically for PlayerData objects
// The codec for PlayerData is passed to the factory method
DataStore<PlayerData> playerStore = provider.create(PlayerData.CODEC);

// Use the store to save or load player data
playerStore.save("player-uuid", playerData);
```

### Anti-Patterns (Do NOT do this)
-   **Implementation-Specific Code:** Do not cast the provider to a concrete type (e.g., FileDataStoreProvider). All interaction must be through the DataStoreProvider interface to maintain storage-layer independence.
-   **Frequent Creation:** The create method may be an expensive operation involving resource allocation. Do not call it repeatedly within a loop or on a hot path. Request a DataStore once during system initialization and cache the reference.
-   **Ignoring The Codec:** Passing an incorrect or null codec to the create method will result in runtime exceptions or data corruption. The provided codec must accurately represent the data type being stored.

## Data Pipeline
The DataStoreProvider is a factory, not a data processor. It sits at the beginning of the data persistence flow, responsible for creating the very components that will later handle the data.

> Flow:
> Server Configuration File -> **DataStoreProvider.CODEC** (Deserializes and selects implementation) -> Game System Initialization -> **DataStoreProvider.create()** -> DataStore Instance -> Game Logic (Read/Write Operations)

