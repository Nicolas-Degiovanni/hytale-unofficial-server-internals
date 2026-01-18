---
description: Architectural reference for PrefabEntry
---

# PrefabEntry

**Package:** com.hypixel.hytale.server.core.prefab
**Type:** Immutable Value Object

## Definition
```java
// Signature
public record PrefabEntry(@Nonnull Path path, @Nonnull Path relativePath, @Nullable AssetPack pack, @Nonnull String displayName) {
```

## Architecture & Concepts
The PrefabEntry record is a lightweight, immutable data carrier that acts as a descriptor for a single prefab asset discovered by the server. Its primary role is to provide a consistent and unified representation of a prefab, regardless of its origin—whether it is a core game asset, part of a third-party asset pack, or a server-specific file.

This class serves as a crucial data-transfer object between the asset discovery layer and various consumer systems. By encapsulating metadata such as file paths and asset pack information, it decouples systems like the world editor, entity spawners, and administrative UIs from the complexities of the underlying file system and asset loading mechanisms. It effectively acts as a handle or a manifest for a prefab before its full data is loaded into memory.

## Lifecycle & Ownership
- **Creation:** Instances of PrefabEntry are created during the server's asset scanning phase. A higher-level manager, such as a hypothetical PrefabService or AssetScanner, is responsible for traversing asset directories, identifying prefab files, and instantiating a PrefabEntry for each one found.

- **Scope:** A PrefabEntry is a transient object. Its lifetime is typically bound to the scope of the collection that holds it, such as a list of available prefabs returned by a service. While these collections may be cached for the server session, individual entries themselves hold no persistent state and are not managed singletons.

- **Destruction:** As a simple record with no external resources, a PrefabEntry is eligible for garbage collection as soon as it is no longer referenced. There is no explicit destruction or cleanup process.

## Internal State & Concurrency
- **State:** **Immutable**. As a Java record, all fields of PrefabEntry are final by definition. Once an instance is created, its state cannot be altered. This guarantees that the prefab's metadata remains consistent throughout its lifecycle.

- **Thread Safety:** **Fully thread-safe**. Due to its immutability, a PrefabEntry instance can be safely shared and read across multiple threads without any need for locks or other synchronization primitives. This makes it ideal for use in concurrent systems, such as parallel asset loading or multi-threaded game logic.

## API Surface
The public contract consists of the record's component accessors and several utility methods for interpreting its data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| path() | Path | O(1) | Returns the absolute, canonical path to the prefab file on disk. |
| relativePath() | Path | O(1) | Returns the path relative to its asset source root. Used as a stable identifier. |
| pack() | AssetPack | O(1) | Returns the AssetPack this prefab belongs to, or null if it is a loose server file. |
| displayName() | String | O(1) | Returns the calculated display name, which may include the asset pack prefix. |
| isFromBasePack() | boolean | O(1) | Checks if the prefab originates from the core game's base asset pack. |
| isFromAssetPack() | boolean | O(1) | Checks if the prefab originates from any asset pack (base or otherwise). |
| getPackName() | String | O(1) | Returns the name of the source AssetPack, or "Server" for loose files. |
| getFileName() | String | O(1) | Extracts the simple file name (e.g., "house.prefab") from the path. |
| getDisplayNameWithPack() | String | O(1) | Returns a formatted string for UI purposes, including the pack name as a prefix. |

## Integration Patterns

### Standard Usage
A PrefabEntry should be obtained from a central service or manager that is responsible for asset discovery. It is then used as a data source to populate user interfaces or as an identifier to request the full loading of the prefab asset.

```java
// Obtain a list of all available prefabs from a central service
List<PrefabEntry> allPrefabs = prefabService.getAvailablePrefabs();

// Populate a UI dropdown with user-friendly names
for (PrefabEntry entry : allPrefabs) {
    // The display name is pre-formatted for UI consumption
    uiComboBox.addOption(entry.getDisplayNameWithPack(), entry.relativePath());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Avoid constructing a PrefabEntry manually with `new PrefabEntry(...)`. These objects are intended to be created by the authoritative asset scanning system to ensure path and asset pack information is canonical and correct. Manual creation can lead to inconsistent or invalid asset references.

- **State Assumption:** Do not assume the underlying file represented by a PrefabEntry still exists or is unchanged. This record is a snapshot in time. Any long-term use should involve re-validating the entry or refreshing it from the source service.

## Data Pipeline
PrefabEntry is a data structure that represents an item *within* a larger data flow. It is the output of the discovery phase and the input for consumption phases.

> Flow:
> Filesystem Scan -> **PrefabEntry (Instantiation)** -> Prefab Registry Cache -> UI Layer / World Generation System -> Asset Loader Request

