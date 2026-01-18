---
description: Architectural reference for Config<T>
---

# Config<T>

**Package:** com.hypixel.hytale.server.core.util
**Type:** Transient Utility

## Definition
```java
// Signature
public class Config<T> {
```

## Architecture & Concepts
The Config class is a generic, stateful utility responsible for managing the lifecycle of a single configuration file on disk. It provides an asynchronous, type-safe abstraction over file I/O and data serialization, decoupling server systems from the underlying persistence mechanism.

Its primary role is to load a JSON file into a specific Java object of type T, cache it in memory, and provide a mechanism to write it back to disk. The class leverages a **BuilderCodec** to handle the complex logic of serialization and deserialization, making the Config class itself focused purely on I/O orchestration and state management.

The design is fundamentally asynchronous, built upon Java's CompletableFuture. This ensures that file operations, which can be slow, do not block critical server threads. Instead, systems can request a configuration load and chain dependent logic to execute once the data is available.

## Lifecycle & Ownership
- **Creation:** A Config instance is created manually whenever a system needs to interact with a specific configuration file. The creator must provide a base directory path, a logical name for the configuration (which becomes the filename), and a corresponding BuilderCodec for the target type T. It is typically instantiated and held as a field within a larger manager service (e.g., a WorldSettingsManager).

- **Scope:** The lifetime of a Config object is bound to its owner. It is designed to be a long-lived object that persists as long as the configuration data it represents is required, often for the entire server session.

- **Destruction:** The object is cleaned up by the Java garbage collector when its owning service is destroyed and all references to it are released. It holds no native resources and does not require an explicit destruction or close method.

## Internal State & Concurrency
- **State:** The internal state is **mutable** and represents the three stages of the configuration's lifecycle: unloaded, loading, and loaded.
    1.  **Unloaded:** The initial state. The internal *config* object is null and the *loadingConfig* future is null.
    2.  **Loading:** The *load* method has been called. The *loadingConfig* future is non-null, representing the ongoing I/O operation. The *config* object is still null.
    3.  **Loaded:** The asynchronous load has completed. The *config* object holds the data, and the *loadingConfig* future is reset to null.

- **Thread Safety:** This class is **conditionally thread-safe** and must be used with caution in multi-threaded environments.
    - The *load* method is idempotent. Multiple concurrent calls will return the same CompletableFuture, preventing redundant file reads.
    - The *get* method is a potential blocking point. If called while a load is in progress, it will call *join* on the future, blocking the calling thread until the I/O is complete. This can lead to performance degradation if called on a performance-critical thread.
    - **WARNING:** There is no explicit locking. Concurrent calls to *save* and *load* can lead to non-deterministic behavior. All interactions with a single Config instance should ideally be coordinated or dispatched via a single-threaded executor to guarantee consistency.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | CompletableFuture<T> | O(N) | Initiates an asynchronous load from disk. Returns a future that completes with the config object. Idempotent. |
| get() | T | O(1) / O(N) | Returns the loaded config object. Throws IllegalStateException if not loaded. Blocks if a load is in progress. |
| save() | CompletableFuture<Void> | O(N) | Initiates an asynchronous save to disk. Returns a future that completes upon completion. Throws IllegalStateException if not loaded. |

*Complexity O(N) refers to the size of the configuration file on disk.*

## Integration Patterns

### Standard Usage
The standard pattern is to initiate the load, attach subsequent logic to the returned future, and only call *get* from within that subsequent logic or after the future is known to be complete. This preserves the non-blocking nature of the system.

```java
// In a service's initialization method
Config<ServerProperties> serverConfig = new Config<>(
    serverRootPath, 
    "server", 
    ServerProperties.CODEC
);

// Asynchronously load and then apply the settings
serverConfig.load().thenAccept(properties -> {
    // This code executes on a worker thread once the file is loaded
    this.applyServerProperties(properties);
    LOGGER.info("Server properties loaded successfully.");
}).exceptionally(error -> {
    LOGGER.error("Failed to load server properties!", error);
    return null;
});

// Later, to access the cached data (after load is complete)
ServerProperties props = serverConfig.get();
```

### Anti-Patterns (Do NOT do this)
- **Blocking Critical Threads:** Do not call *get* on the main server thread without first ensuring the *load* future has completed. This will freeze the server while it waits for disk I/O.

  ```java
  // BAD: This will block the current thread
  Config<WorldData> worldConfig = ...
  worldConfig.load(); // Kicks off async load
  WorldData data = worldConfig.get(); // Immediately blocks until load is finished
  ```

- **Assuming Immediate Availability:** Do not call *get* or *save* immediately after instantiation. The object is in an unloaded state and will throw an IllegalStateException.

  ```java
  // BAD: Throws IllegalStateException
  Config<WorldData> worldConfig = new Config<>(...);
  WorldData data = worldConfig.get(); 
  ```

- **Ignoring Save Future:** Saving is also asynchronous. Logic that depends on the file being successfully written to disk must be chained to the future returned by the *save* method.

## Data Pipeline

The flow of data during a **load** operation is as follows. The Config class orchestrates the interaction between the filesystem, the codec, and the application code.

> Flow:
> JSON File on Disk → Filesystem Read → RawJsonReader → BuilderCodec (Deserialization) → **Config<T>** (Internal State Cache) → Application via get()

