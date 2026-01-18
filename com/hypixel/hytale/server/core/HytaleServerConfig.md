---
description: Architectural reference for HytaleServerConfig
---

# HytaleServerConfig

**Package:** com.hypixel.hytale.server.core
**Type:** Singleton-scoped Data Object

## Definition
```java
// Signature
public class HytaleServerConfig {
```

## Architecture & Concepts

The HytaleServerConfig class is the authoritative, in-memory representation of the server's configuration, loaded from the `config.json` file. It serves as the central source of truth for all server parameters, from basic properties like the server name to complex, nested configurations for modules, plugins, and network behavior.

Architecturally, this class is not merely a passive data container. It is an active component in the server's lifecycle, built upon a sophisticated serialization and deserialization framework, **Hytale Codec**.

The core of its design is the static `CODEC` field, an instance of `BuilderCodec`. This declarative API defines a bidirectional mapping between the JSON structure on disk and the Java fields of the HytaleServerConfig object. Key features of this design include:

*   **Versioning:** The codec is versioned (`codecVersion(3)`), allowing for graceful upgrades and backward compatibility with older `config.json` formats. The handling of `legacyPluginConfig` demonstrates this, where old "Plugins" keys are transparently migrated to the new "Mods" structure post-decode.
*   **Type Safety:** The codec system maps JSON primitives to specific Java types (String, Integer, Duration, etc.), ensuring type safety at the boundary between the file system and the running server.
*   **Extensibility via Modules:** The nested `Module` class provides a powerful pattern for extensibility. It acts as a generic configuration container, allowing different server systems or plugins to store arbitrary BSON data under a unique key. This avoids polluting the top-level configuration schema with domain-specific fields, promoting loose coupling.
*   **Dirty State Tracking:** The class implements a "dirty flag" pattern using an `AtomicBoolean` named `hasChanged`. Every setter method calls `markChanged()`, which flips this flag. This allows the server to efficiently determine if the configuration needs to be persisted back to disk, avoiding unnecessary I/O operations.

The configuration is structured hierarchically using nested static classes (`Defaults`, `ConnectionTimeouts`, `RateLimitConfig`) to logically group related settings.

### Lifecycle & Ownership
-   **Creation:** The HytaleServerConfig object is instantiated exclusively through the static `load(Path path)` method, typically called once during server bootstrap. If `config.json` does not exist, a new instance with default values is created and immediately scheduled for saving to disk.
-   **Scope:** The object has a global, session-wide scope. It is created at server startup and persists in memory for the entire lifetime of the server process. It is expected to be held as a final field in the main server or application context class.
-   **Destruction:** The object is garbage collected upon server shutdown when the JVM terminates. Changes are persisted to disk by explicitly calling the static `save()` method, which is often tied to a shutdown hook or an administrative command.

## Internal State & Concurrency
-   **State:** The object's state is highly **mutable**. Nearly all configuration values can be changed at runtime through public setters. It is a live representation of the server's configuration, not an immutable snapshot. The state includes both strongly-typed fields and a map of generic `Module` objects for extensible data.
-   **Thread Safety:** The class is **not fully thread-safe** and requires careful management in a concurrent environment.
    -   Collections for `modules`, `logLevels`, and `modConfig` use `ConcurrentHashMap`, making structural modifications (like adding a new module) safe across threads.
    -   The `hasChanged` flag uses `AtomicBoolean`, which is safe for concurrent updates.
    -   **WARNING:** Individual, non-collection fields (e.g., `serverName`, `maxPlayers`) are not protected by locks or volatiles. Concurrent writes from multiple threads to these fields can lead to race conditions and memory visibility issues. It is assumed that writes will be synchronized externally or performed exclusively on a main server thread. Reads are generally safe, but may not see the most recent write from another thread without a happens-before relationship.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load(Path path) | HytaleServerConfig | I/O Bound | **(Static)** Deserializes config from disk. Creates a default config if file is missing. |
| save(HytaleServerConfig config) | CompletableFuture<Void> | I/O Bound | **(Static)** Asynchronously serializes the config object to disk. |
| getModule(String moduleName) | Module | O(1) | Retrieves or creates a configuration container for a specific module. |
| markChanged() | void | O(1) | Sets the internal dirty flag, indicating the config should be saved. |
| consumeHasChanged() | boolean | O(1) | Atomically retrieves the dirty flag's current state and resets it to false. |

## Integration Patterns

### Standard Usage

The configuration should be loaded once at startup and passed to dependent services. Modifications should be followed by a call to `save`.

```java
// During server initialization
HytaleServerConfig config = HytaleServerConfig.load();

// Accessing a simple property
int maxPlayers = config.getMaxPlayers();

// Accessing a nested, module-specific property
HytaleServerConfig.Module myModuleConfig = config.getModule("myCustomModule");
boolean isFeatureEnabled = myModuleConfig.isEnabled(true); // default to true

// Modifying a property at runtime (e.g., via a command)
config.setServerName("My New Server Name!");
HytaleServerConfig.save(config).join(); // Persist the change
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new HytaleServerConfig()`. This bypasses the file loading mechanism and creates a transient, default configuration that will not be loaded from or saved to `config.json`. Always use the static `HytaleServerConfig.load()` method.
-   **Frequent Saving:** Do not call `save()` after every single modification in a tight loop. Batch changes where possible and save periodically or on shutdown. The `consumeHasChanged()` method can be used to build an efficient auto-save system.
-   **Unsynchronized Concurrent Writes:** Do not call setters for simple properties like `setServerName` or `setMaxPlayers` from multiple threads simultaneously without external synchronization. This can lead to lost updates and undefined behavior.

## Data Pipeline

The HytaleServerConfig acts as a bridge between the file system and the server's runtime components.

**Loading Pipeline:**
> Flow:
> `config.json` (File) -> Filesystem Read -> JSON String -> `RawJsonReader` -> `HytaleServerConfig.CODEC` (Deserialization) -> **`HytaleServerConfig` Instance** -> Server Systems

**Saving Pipeline:**
> Flow:
> Server System -> `config.set...()` -> `config.markChanged()` -> `HytaleServerConfig.save()` -> `HytaleServerConfig.CODEC` (Serialization) -> `BsonDocument` -> `BsonUtil` -> `config.json` (File)

