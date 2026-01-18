---
description: Architectural reference for DiskDataStoreProvider
---

# DiskDataStoreProvider

**Package:** com.hypixel.hytale.server.core.universe.datastore
**Type:** Transient Factory

## Definition
```java
// Signature
public class DiskDataStoreProvider implements DataStoreProvider {
```

## Architecture & Concepts
The DiskDataStoreProvider is a concrete implementation of the **Strategy Pattern**, conforming to the DataStoreProvider interface. Its sole responsibility is to act as a factory for creating DataStore instances that persist data to a local file system.

This class serves as a critical abstraction layer that decouples the server's data management logic from the physical storage medium. High-level systems, such as a Universe or World, are configured to use a DataStoreProvider without needing to know its specific implementation. This allows server administrators to switch the storage backend (e.g., from Disk to a database) purely through configuration, without any code changes.

The static CODEC field is the primary mechanism for its integration into the server. The server's configuration loader uses this codec to deserialize settings from a configuration file (e.g., a JSON or YAML file) into a fully configured DiskDataStoreProvider instance. The provider is identified by its static ID "Disk".

## Lifecycle & Ownership
- **Creation:** Instances are not created manually. They are instantiated by the server's configuration and codec system during the bootstrap phase. The system reads a configuration source, finds a section specifying the "Disk" provider type, and uses the public static CODEC to construct an instance, populating the internal path field.
- **Scope:** The lifetime of a DiskDataStoreProvider is tied to the component that owns it, such as a specific World instance. It is not a global singleton. It persists for the duration of the owning component's lifecycle.
- **Destruction:** The object is marked for garbage collection when its owner (e.g., the World) is unloaded and all references to it are released. It does not manage any unmanaged resources and has no explicit destruction or `close` method.

## Internal State & Concurrency
- **State:** The internal state is minimal, consisting of a single `path` string. This state is effectively **immutable** after the object has been constructed by the codec system.
- **Thread Safety:** This class is **thread-safe**. Its internal state is immutable, and its primary factory method, `create`, is re-entrant as it only instantiates new objects and does not modify its own state.

**WARNING:** While the provider itself is thread-safe, the DataStore instances it creates are responsible for their own concurrency control. The resulting DiskDataStore objects perform file I/O and are likely **not** thread-safe. Access to a DataStore instance from multiple threads must be externally synchronized.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| create(BuilderCodec<T> builderCodec) | DataStore<T> | O(1) | Factory method. Instantiates and returns a new DiskDataStore configured to read/write data at the provider's file system path. |

## Integration Patterns

### Standard Usage
A developer should almost never interact with this class directly. Instead, it is configured at the server level and used by higher-level data management services.

```java
// Hypothetical usage within a World management system
// The 'provider' is injected or configured, not instantiated here.
DataStoreProvider provider = worldConfig.getDataStoreProvider();

// The system requests a store for PlayerProfile data
DataStore<PlayerProfile> playerProfileStore = provider.create(PlayerProfile.CODEC);

// The system can now use the store to save and load data
playerProfileStore.save("player-uuid", somePlayerProfile);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new DiskDataStoreProvider("/path/to/data")` within application logic. This hardcodes the storage implementation and path, defeating the purpose of the configurable provider model. All provider configuration should exist in server configuration files.
- **State Modification:** Do not use reflection or other means to modify the internal `path` field after construction. The provider and the stores it creates assume this path is constant for their entire lifetime.

## Data Pipeline
The DiskDataStoreProvider is not part of a data processing pipeline. Instead, it is a factory that *enables* a data pipeline. Its role is to produce the endpoint for persistence.

> Flow:
> Server Configuration File -> Hytale Codec System -> **DiskDataStoreProvider Instance** -> `create()` Invocation -> DiskDataStore Instance -> File System I/O

---

