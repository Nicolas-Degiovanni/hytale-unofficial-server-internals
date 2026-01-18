---
description: Architectural reference for AssetTypeRegistry
---

# AssetTypeRegistry

**Package:** com.hypixel.hytale.builtin.asseteditor
**Type:** Singleton Service

## Definition
```java
// Signature
public class AssetTypeRegistry {
```

## Architecture & Concepts
The AssetTypeRegistry is the central authority for asset type definitions within the server-side Asset Editor. It functions as a dynamic service locator, mapping file paths and unique string identifiers to their corresponding `AssetTypeHandler` implementations.

This class is the cornerstone of the Asset Editor's extensible architecture. It decouples the generic asset management framework from the concrete logic required to handle specific asset types like models, textures, or configuration files. By registering different `AssetTypeHandler` implementations, the system can be extended to support new types of assets without modifying the core editor engine.

Its primary responsibilities are:
1.  **Registration:** Maintain a live, thread-safe collection of all available asset handlers.
2.  **Resolution:** Determine the correct handler for a given file path based on its directory and file extension.
3.  **Configuration Caching:** Pre-compile and cache a network packet that describes all registered asset types, which is sent to clients upon connection to initialize their user interface.

## Lifecycle & Ownership
-   **Creation:** A single instance of AssetTypeRegistry is instantiated by the server's main service container when the Asset Editor module is initialized.
-   **Scope:** The registry is a long-lived singleton. It persists for the entire lifecycle of the server process, holding the global state for asset type definitions.
-   **Destruction:** The object is eligible for garbage collection only upon server shutdown or a complete unload of the Asset Editor module. There is no explicit public destruction or cleanup method.

## Internal State & Concurrency
-   **State:** The internal state is mutable. The core state is the `assetTypeHandlers` map, which is populated and modified at runtime via registration and unregistration calls. It also maintains a cached `setupPacket` for network transmission.

-   **Thread Safety:** This class is designed to be **thread-safe**. The primary state is stored in a `ConcurrentHashMap`, allowing for safe concurrent reads, writes, and iterations. The `registerAssetType` method uses the atomic `putIfAbsent` operation to prevent race conditions during registration. This is critical in an environment where network threads may be resolving asset types while a main or plugin thread is registering new ones.

    **Warning:** While the collection itself is thread-safe, the `setupPacket` field is not. It is read and written non-atomically. It is imperative that `setupPacket()` is called from a single, controlled thread after all registrations are complete and before any clients are sent the configuration.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerAssetType(handler) | void | O(1) | Atomically registers a new handler. Throws IllegalArgumentException if an ID is already registered. |
| getAssetTypeHandler(id) | AssetTypeHandler | O(1) | Retrieves a handler by its unique string identifier. |
| getAssetTypeHandlerForPath(path) | AssetTypeHandler | O(N) | Resolves a handler by iterating through all registrations. Returns the first match for file extension and root path. |
| tryGetAssetTypeHandler(path, client, token) | AssetTypeHandler | O(N) | A safe wrapper for path resolution. On failure, automatically sends an error packet to the specified client. |
| setupPacket() | void | O(N) | Compiles the current state of registered handlers into a network packet and caches it internally. **Must be called** after registration is complete. |
| sendPacket(client) | void | O(1) | Sends the cached setup packet to a client. Will fail if `setupPacket()` has not been called. |

## Integration Patterns

### Standard Usage
The registry is intended to be configured during server initialization. All handlers are registered, the setup packet is generated once, and then the registry is used for read-only resolution by network threads.

```java
// During server bootstrap or module loading
AssetTypeRegistry registry = serviceContainer.get(AssetTypeRegistry.class);

// Register all known asset handlers
registry.registerAssetType(new ModelAssetHandler());
registry.registerAssetType(new TextureAssetHandler());
registry.registerAssetType(new SoundAssetHandler());

// Bake the configuration into a reusable network packet
registry.setupPacket();

// ... later, when a client connects to the Asset Editor
EditorClient client = getClient();
registry.sendPacket(client);

// ... later, when handling a client request for a specific file
Path assetPath = Path.of("assets/models/character.hymodel");
AssetTypeHandler handler = registry.tryGetAssetTypeHandler(assetPath, client, requestToken);
if (handler != null) {
    handler.processEdit(assetPath);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new AssetTypeRegistry()`. This creates an orphaned instance that is not integrated with the server's service architecture, leading to a system with no registered asset types. Always retrieve the shared instance from the central service container.

-   **Delayed Packet Setup:** Do not call `sendPacket` before `setupPacket` has been successfully executed. This will result in a NullPointerException or the transmission of stale data. The `setupPacket` method is not idempotent and should be called after any modification to the handler set.

-   **Frequent Re-Registration:** The path resolution methods (`getAssetTypeHandlerForPath`) have O(N) complexity. While acceptable for a moderate number of asset types, frequently registering and unregistering hundreds of handlers at runtime can degrade path resolution performance.

## Data Pipeline
The registry participates in two primary data flows: client configuration and request processing. The request processing flow is the most common.

> Flow:
> Client Edit Request (Packet) -> Server Network Layer -> **AssetTypeRegistry**.tryGetAssetTypeHandler -> Resolved AssetTypeHandler -> Asset-Specific Business Logic

