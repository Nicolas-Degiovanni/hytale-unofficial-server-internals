---
description: Architectural reference for AssetRegistryLoader
---

# AssetRegistryLoader

**Package:** com.hypixel.hytale.server.core.asset
**Type:** Utility

## Definition
```java
// Signature
public class AssetRegistryLoader {
```

## Architecture & Concepts

The AssetRegistryLoader is a static utility class that serves as the primary orchestrator for the server's entire asset loading pipeline. It is not an asset manager itself; rather, it is a stateless, procedural driver that operates on the global, static AssetRegistry. Its core responsibility is to discover, read, decode, and populate all game assets from disk into their final in-memory representations within the various AssetStore instances managed by the AssetRegistry.

This class embodies a critical architectural pattern: the separation of the loading process from the asset storage. The AssetRegistryLoader handles the *how* and *when* of loading, while the AssetRegistry handles the *what* and *where* of storage.

The loading process is deterministic and dependency-aware, managed by the AssetStoreIterator. This iterator performs a topological sort on the dependency graph defined in the AssetRegistryLoader's static initializer block. This guarantees that foundational assets (e.g., SoundEvent) are fully loaded before dependent assets (e.g., BlockSoundSet) attempt to reference them, preventing cascading failures and ensuring system stability.

The loader operates in two distinct phases:
1.  **Pre-loading:** The `preLoadAssets` method handles the injection of hardcoded, engine-level default assets (e.g., `BlockType.UNKNOWN`, `AmbienceFX.EMPTY`). These assets are essential for the engine to function correctly even before any game data is loaded.
2.  **Main Loading:** The `loadAssets` method processes an AssetPack, traversing its directory structure to load all discoverable game assets.

Furthermore, this class contains essential tooling for content creators, including the ability to introspect all registered asset types and generate comprehensive JSON Schemas for validation and IDE autocompletion.

## Lifecycle & Ownership

-   **Creation:** As a static utility class, AssetRegistryLoader is never instantiated. Its methods and fields are loaded by the JVM's ClassLoader at server startup and exist in a static context.
-   **Scope:** The class and its static methods are globally accessible and persist for the entire lifetime of the server application.
-   **Destruction:** The class is unloaded only when the Java Virtual Machine shuts down. No manual cleanup is required.

## Internal State & Concurrency

-   **State:** AssetRegistryLoader is fundamentally **stateless**. It does not contain any instance or static fields that hold mutable data related to the loading process. All state is externalized and managed by the static AssetRegistry and its constituent AssetStore objects.

-   **Thread Safety:** The core loading and synchronization methods are **thread-safe** through explicit locking.
    -   `preLoadAssets`, `loadAssets`, and `sendAssets` all acquire a write lock on the global `AssetRegistry.ASSET_LOCK`.
    -   This design serializes all major asset operations, ensuring that the asset database remains consistent. It is a deliberate trade-off, prioritizing data integrity over concurrent performance during the critical, one-off phases of startup and client synchronization.
    -   **WARNING:** Attempting to read from the AssetRegistry on another thread while a loading operation is in progress will result in that thread blocking until the write lock is released.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| preLoadAssets(LoadAssetEvent) | static void | O(S log S) | Phase 1 loading. Iterates the dependency graph of S stores to load engine-default assets. Blocks until complete. |
| loadAssets(LoadAssetEvent, AssetPack) | static void | O(F) | Phase 2 loading. Loads all F asset files from a given AssetPack. Blocks until complete. |
| sendAssets(PacketHandler) | static void | O(A) | Serializes all A loaded assets into network packets and sends them to a client. Blocks until complete. |
| generateSchemas(...) | static Map | O(S) | **Tooling Only.** Introspects S asset stores to generate in-memory JSON Schema definitions. |
| writeSchemas(LoadAssetEvent) | static void | O(S) | **Tooling Only.** Triggers schema generation, writes files to disk, and initiates a server shutdown. |

## Integration Patterns

### Standard Usage

The AssetRegistryLoader is invoked by the server's primary bootstrap sequence. The order of operations is critical and must be respected.

```java
// During server startup, typically within an AssetModule
AssetPack basePack = AssetModule.get().getBaseAssetPack();
LoadAssetEvent event = new LoadAssetEvent();

// Phase 1: Load engine defaults
AssetRegistryLoader.preLoadAssets(event);

// Phase 2: Load all game assets from the primary asset pack
AssetRegistryLoader.loadAssets(event, basePack);

// ... server is now running ...

// When a player joins, synchronize their client
PacketHandler playerPacketHandler = player.getPacketHandler();
AssetRegistryLoader.sendAssets(playerPacketHandler);
```

### Anti-Patterns (Do NOT do this)

-   **Incorrect Invocation Order:** Never call `loadAssets` before `preLoadAssets`. The pre-loading phase establishes critical default assets that the main loading phase may depend on. Failure to adhere to this order will result in unpredictable `NullPointerException`s and data corruption.
-   **Concurrent Modification:** Do not attempt to manually register new AssetStores with the AssetRegistry while `loadAssets` is executing. The collection of stores is considered immutable during the loading process.
-   **Runtime Schema Generation:** Do not invoke `writeSchemas` on a production server. This method is designed exclusively for development and build pipelines. It forces a server shutdown upon completion by design.

## Data Pipeline

The AssetRegistryLoader acts as the central processing unit in several key data flows.

> **Asset Loading Flow:**
> Filesystem (`.json`, `.particlesystem`, etc.) → AssetPack → **AssetRegistryLoader.loadAssets** → AssetStoreIterator → AssetCodec → In-Memory Asset Object → AssetRegistry

> **Client Synchronization Flow:**
> AssetRegistry → **AssetRegistryLoader.sendAssets** → HytaleAssetStore → AssetPacketGenerator → Packet → PacketHandler → Network

> **Schema Generation Flow (Tooling):**
> AssetRegistry (AssetStore Codecs) → **AssetRegistryLoader.generateSchemas** → SchemaContext → In-Memory Schema Object → BsonUtil → Filesystem (`Schema/*.json`)

