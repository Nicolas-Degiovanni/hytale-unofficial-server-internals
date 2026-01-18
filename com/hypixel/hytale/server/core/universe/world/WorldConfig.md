---
description: Architectural reference for WorldConfig
---

# WorldConfig

**Package:** com.hypixel.hytale.server.core.universe.world
**Type:** Transient

## Definition
```java
// Signature
public class WorldConfig {
```

## Architecture & Concepts

The WorldConfig class is the authoritative data model for all persistent and runtime settings of a single game world. It serves as the central source of truth that dictates the behavior of nearly every major server system, from world generation and chunk storage to gameplay rules like PvP and NPC spawning.

Architecturally, this class is designed around Hytale's powerful **Codec** system. The static `CODEC` field, an instance of `BuilderCodec`, is the cornerstone of this design. It declaratively defines the entire serialization schema for a world's configuration, including:

*   **Versioning:** The schema is versioned, allowing for graceful upgrades and backward compatibility for older world files.
*   **Schema Definition:** Each configurable property is explicitly mapped to a persistent key (e.g., "DisplayName", "Seed") and its corresponding data type.
*   **Extensibility:** It provides a dedicated `Plugin` configuration map, allowing third-party modules to store their own world-specific data within the same configuration file without modifying the core class.
*   **Validation:** The schema includes validators to ensure data integrity, such as checking for non-null values or referencing valid asset identifiers.

This codec-centric approach decouples the in-memory object representation from its on-disk format (BSON/JSON), enabling robust loading, saving, and data migration. The class acts as a bridge between the persistent world data on disk and the live, operational `World` object in the server's memory.

## Lifecycle & Ownership

-   **Creation:** A WorldConfig instance is brought into existence in one of two ways:
    1.  **Deserialization (Standard):** The primary creation path is through the static `WorldConfig.load(Path)` method. This asynchronous operation reads a configuration file from disk, decodes it using the master `CODEC`, and constructs a fully populated WorldConfig object. This occurs when the server's `Universe` loads a world.
    2.  **Direct Instantiation (New Worlds):** `new WorldConfig()` is used to create a configuration for a brand new world with default values. The public constructor immediately calls `markChanged()`, flagging this new instance for its initial save to disk.

-   **Scope:** The lifecycle of a WorldConfig object is tightly bound to its parent `World` object. It is loaded into memory when the world is loaded and remains for the entire duration that the world is active on the server.

-   **Destruction:** The object is eligible for garbage collection once the corresponding `World` is unloaded and all references to the configuration are released. There is no explicit destruction or cleanup method. Persistence is managed separately via the static `save` method, which should be called before the world is unloaded if changes have been made.

## Internal State & Concurrency

-   **State:** The WorldConfig object is highly **mutable**. It is a container for the dynamic state of a world's settings, which can be altered at runtime via server commands or game logic. A critical piece of internal state is the transient `hasChanged` `AtomicBoolean` flag. This flag is used as a "dirty" marker to signal that the in-memory state has diverged from the on-disk state and requires saving.

-   **Thread Safety:** This class is **not thread-safe**. While the `hasChanged` flag uses an atomic type, this only guarantees atomicity for the flag itself. The vast majority of fields are accessed and mutated without any synchronization.

    **WARNING:** All mutations to a WorldConfig instance must be performed from a single, controlled thread, typically the main server thread or the dedicated thread for the world it belongs to. Concurrent modification from multiple threads will result in race conditions, data corruption, and unpredictable server behavior. The static `load` and `save` methods are asynchronous and offload I/O from the main thread, but the resulting object must be handled in a thread-safe manner.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load(Path path) | CompletableFuture<WorldConfig> | O(I/O) | **Static.** Asynchronously loads and deserializes a WorldConfig from a given file path. This is the primary entry point for loading existing worlds. |
| save(Path path, WorldConfig config) | CompletableFuture<Void> | O(I/O) | **Static.** Asynchronously serializes and writes a WorldConfig instance to a given file path. |
| markChanged() | void | O(1) | Sets the internal "dirty" flag to true, indicating the configuration needs to be saved. |
| consumeHasChanged() | boolean | O(1) | Atomically retrieves the current state of the "dirty" flag and resets it to false. Essential for save-on-demand logic. |
| getPluginConfig() | MapKeyMapCodec.TypeMap | O(1) | Returns the map used for storing custom data for plugins. |

## Integration Patterns

### Standard Usage

The WorldConfig is held as a member variable within a `World` object. Other server systems query this object to guide their behavior. A persistence manager is responsible for periodically checking the dirty flag and saving the configuration to disk.

```java
// During world loading
WorldConfig config = WorldConfig.load(worldPath).join();
World myWorld = new World(config);

// Later, in a game logic update or command handler
WorldConfig currentConfig = myWorld.getConfig();
currentConfig.setPvpEnabled(true);
currentConfig.markChanged(); // CRITICAL: Caller must mark the object as dirty.

// In a separate persistence system
if (myWorld.getConfig().consumeHasChanged()) {
    WorldConfig.save(worldPath, myWorld.getConfig());
}
```

### Anti-Patterns (Do NOT do this)

-   **Mutating without Marking:** Modifying properties via setters without subsequently calling `markChanged()` will result in lost data. The changes will exist in memory but will never be persisted to disk, as the persistence system will not know the object is dirty.
-   **Concurrent Modification:** As stated previously, reading from and writing to the same WorldConfig instance from multiple threads without external locking is strictly forbidden and will lead to a corrupted state.
-   **Ignoring Codec Defaults:** When creating a new world, do not manually set every property. Rely on the defaults provided by the `CODEC` and the `WorldConfig` constructor, then override only what is necessary.

## Data Pipeline

The WorldConfig class is a critical node in the data flow between the server's persistent storage and its live runtime environment.

> **Loading Flow:**
> Disk File (world.json) → `RawJsonReader` → `WorldConfig.CODEC` (Deserialization) → **In-memory WorldConfig Instance** → `World` Object

> **Saving Flow:**
> Runtime Modification → `config.set...()` → `config.markChanged()` → Persistence Manager detects change via `consumeHasChanged()` → `WorldConfig.CODEC` (Serialization) → BsonDocument → **Disk File (world.json)**

