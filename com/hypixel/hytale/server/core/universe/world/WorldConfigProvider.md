---
description: Architectural reference for WorldConfigProvider
---

# WorldConfigProvider

**Package:** com.hypixel.hytale.server.core.universe.world
**Type:** Provider Interface

## Definition
```java
// Signature
public interface WorldConfigProvider {
    // Default methods for loading and saving

    public static class Default implements WorldConfigProvider {
    }
}
```

## Architecture & Concepts
The WorldConfigProvider interface defines a strategic contract for the persistence of WorldConfig objects. It acts as a crucial abstraction layer between the world management system and the underlying storage medium, decoupling the server core from the specifics of file I/O, database transactions, or other persistence mechanisms.

Its primary architectural role is to govern the serialization and deserialization of a world's fundamental configuration. The interface is designed for asynchronous, non-blocking operations, utilizing CompletableFuture to prevent I/O-bound tasks from stalling the main server threads.

The default implementation provided within the interface handles file-based persistence to a JSON file. Notably, it includes legacy support for migrating older BSON-based configuration files to the current JSON format, demonstrating its role in managing data format evolution across engine versions.

## Lifecycle & Ownership
- **Creation:** Implementations of WorldConfigProvider, such as the provided Default class, are typically instantiated by a higher-level service responsible for world lifecycle management, like a WorldManager or UniverseService. They are not intended to be managed directly by game logic.
- **Scope:** The lifetime of a provider instance is generally transient, created on-demand for a specific load or save operation. As the Default implementation is stateless, a single instance could be reused, but the typical pattern is to treat it as a short-lived utility.
- **Destruction:** Instances are eligible for garbage collection once the CompletableFuture they return is completed and no longer referenced. There is no explicit destruction or cleanup method required.

## Internal State & Concurrency
- **State:** The WorldConfigProvider interface is inherently stateless. The Default implementation is also stateless, containing no member variables. All necessary data is passed as arguments to its methods.
- **Thread Safety:** The interface and its Default implementation are thread-safe. Methods can be invoked from any thread. The use of CompletableFuture ensures that the I/O operations are executed asynchronously, and the results are safely published back to the calling thread or a subsequent stage in a reactive pipeline.

**WARNING:** While the provider itself is thread-safe, the WorldConfig object it produces is not guaranteed to be. Consumers are responsible for ensuring thread-safe access to the resulting configuration object.

## API Surface
The public contract is designed for asynchronous I/O operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load(savePath, name) | CompletableFuture<WorldConfig> | I/O Bound | Asynchronously loads a WorldConfig from the specified path. Handles the migration of legacy config.bson files to config.json. |
| save(savePath, config, world) | CompletableFuture<Void> | I/O Bound | Asynchronously saves the provided WorldConfig to the specified path. The world parameter is unused in the default implementation but is available for more advanced providers. |

## Integration Patterns

### Standard Usage
The provider should be used by a service layer to abstract away the details of world configuration persistence. The asynchronous nature of the API should be respected by chaining dependent actions rather than blocking.

```java
// A world management service using the provider
WorldConfigProvider provider = new WorldConfigProvider.Default();
Path worldSavePath = Paths.get("./worlds/myworld");

provider.load(worldSavePath, "myworld").thenAccept(config -> {
    // Logic to apply the loaded configuration
    System.out.println("World config loaded: " + config.getName());
}).exceptionally(ex -> {
    // Handle I/O errors or deserialization failures
    log.error("Failed to load world config", ex);
    return null;
});
```

### Anti-Patterns (Do NOT do this)
- **Blocking the Main Thread:** Never call get() or join() on the returned CompletableFuture from a performance-critical thread, such as the main server tick loop. This defeats the purpose of the asynchronous design and will cause server stalls.

    ```java
    // BAD: This will block the current thread until I/O is complete.
    WorldConfig config = provider.load(path, name).get();
    ```

- **Ignoring Failure Cases:** The load and save operations can fail due to I/O exceptions or data corruption. Failing to attach an exception handler (e.g., via exceptionally) will result in silent failures and potentially a corrupt server state.

- **Custom Implementation without Migration:** If creating a custom implementation of WorldConfigProvider, failing to replicate the BSON-to-JSON migration logic from the default load method will break compatibility with worlds created on older server versions.

## Data Pipeline
The component orchestrates the flow of configuration data between the file system and in-memory objects.

> **Load Flow:**
> File System (config.json or legacy config.bson) -> Asynchronous File Read -> **WorldConfigProvider.load** -> JSON/BSON Deserialization -> In-Memory WorldConfig Object

> **Save Flow:**
> In-Memory WorldConfig Object -> **WorldConfigProvider.save** -> JSON Serialization -> Asynchronous File Write -> File System (config.json)

