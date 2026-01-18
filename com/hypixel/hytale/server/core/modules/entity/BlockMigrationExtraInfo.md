---
description: Architectural reference for BlockMigrationExtraInfo
---

# BlockMigrationExtraInfo

**Package:** com.hypixel.hytale.server.core.modules.entity
**Type:** Transient

## Definition
```java
// Signature
public class BlockMigrationExtraInfo extends ExtraInfo {
```

## Architecture & Concepts
BlockMigrationExtraInfo is a specialized Data Transfer Object (DTO) designed to carry version-specific data transformation logic into the core data serialization and deserialization pipeline. It extends the base ExtraInfo class, signaling its role as optional, contextual information for a primary operation, typically world or entity loading.

The core architectural pattern employed here is the **Strategy Pattern**. Instead of embedding version-specific migration logic directly within world loaders or codecs, this class encapsulates the migration logic as a `Function`. This decouples the generic data processing system from the concrete, and often complex, rules required to upgrade data from one version to another.

Its primary responsibility is to provide a function that can translate legacy block identifiers (strings) into their modern equivalents during the loading of older world data. This ensures backward compatibility without polluting the primary data loading systems with conditional version checks.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a higher-level system, such as a WorldLoader or ChunkSerializer, immediately before a deserialization operation. The creator is responsible for detecting the data version and supplying the correct migration function for that version.
- **Scope:** The object's lifetime is ephemeral and strictly bound to the scope of the single operation it services. It is created, passed as a parameter, used, and then immediately becomes eligible for garbage collection.
- **Destruction:** Managed entirely by the Java Garbage Collector. There is no explicit cleanup or resource release required.

## Internal State & Concurrency
- **State:** Immutable. The internal `blockMigration` function is a final field, assigned once during construction. The object's state cannot be modified after instantiation.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. However, the thread safety of the operation depends entirely on the provided `Function`. If the supplied function is stateless and free of side effects (e.g., a method reference to a static utility or a simple lambda), the entire construct is safe for concurrent use.

**WARNING:** The system that constructs this object is responsible for ensuring the provided function is thread-safe if the data loading process is multi-threaded.

## API Surface
The public contract is minimal, focused on retrieving the encapsulated migration logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getBlockMigration() | Function | O(1) | Returns the configured block migration function. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by most game logic developers. It is an internal component of the world data loading pipeline. A system responsible for loading chunks would use it to provide context to a generic codec.

```java
// Hypothetical usage within a world loading system
int dataVersion = readDataVersionFromStream(stream);
WorldFormat format = worldFormatRegistry.getFormat(dataVersion);

// The registry provides the correct migration logic for the detected version
Function<String, String> migrationLogic = format.getBlockMigrationFunction();

// The ExtraInfo object is created to pass this logic into the codec
BlockMigrationExtraInfo context = new BlockMigrationExtraInfo(dataVersion, migrationLogic);

// The codec uses the function provided in the context to translate block IDs
ChunkData chunk = chunkCodec.decode(stream, context);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Functions:** Do not provide a `Function` that depends on or modifies mutable external state. This breaks the immutability contract and can introduce severe, non-deterministic bugs in a multi-threaded environment.
- **Incorrect Versioning:** Instantiating this object with a version number that does not correspond to the logic within the provided `Function` will lead to silent data corruption. The version and the function logic must be in sync.
- **Generic Function Carrier:** Do not use this class as a generic container for passing functions outside of the data codec and world loading systems. Its purpose is strictly tied to the ExtraInfo framework for data serialization.

## Data Pipeline
BlockMigrationExtraInfo does not process data itself; rather, it *configures* a step within the data pipeline. It injects behavior into the deserializer.

> Flow:
> Raw Chunk Data (Old Version) -> Chunk Deserializer -> **BlockMigrationExtraInfo (as context)** -> Migrated Block Data (New Version) -> In-Memory Chunk Object

