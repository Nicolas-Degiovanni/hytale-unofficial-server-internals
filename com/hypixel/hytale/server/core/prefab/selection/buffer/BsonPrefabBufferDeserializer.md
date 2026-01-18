---
description: Architectural reference for BsonPrefabBufferDeserializer
---

# BsonPrefabBufferDeserializer

**Package:** com.hypixel.hytale.server.core.prefab.selection.buffer
**Type:** Singleton

## Definition
```java
// Signature
public class BsonPrefabBufferDeserializer implements PrefabBufferDeserializer<BsonDocument> {
```

## Architecture & Concepts
The BsonPrefabBufferDeserializer is a specialized data translation component responsible for converting a persistent prefab representation into a runtime-optimized memory structure. It serves as the bridge between the on-disk BSON format of a Hytale prefab file and the in-memory PrefabBuffer used by the server to instantiate structures in the game world.

This class is a critical part of the asset loading pipeline for world prefabs. Its primary responsibilities include:

*   **Data Format Unmarshalling:** Parsing the hierarchical BsonDocument structure into a flat, column-based buffer.
*   **Version Compatibility:** It actively checks the prefab's version number and can reject outdated or future-versioned assets, preventing world corruption.
*   **Data Migration:** It orchestrates the migration of legacy block IDs by chaining together BlockMigration assets. This ensures that prefabs created with older versions of the game can be loaded correctly in newer versions.
*   **Coordinate System Translation:** It transforms the absolute world coordinates stored in the BSON file into a relative coordinate system based on the prefab's anchor point. This makes the resulting PrefabBuffer position-independent and ready for placement anywhere in the world.

This component is designed to be a pure, stateless function. It does not interact with the game loop, world state, or network layer. Its sole purpose is to perform a single, complex data transformation.

### Lifecycle & Ownership
- **Creation:** The singleton INSTANCE is created by the JVM class loader upon the first access to the BsonPrefabBufferDeserializer class. It is not managed by a dependency injection container or created on-demand.
- **Scope:** As a stateless singleton, its lifecycle is tied to the server process itself. It persists for the entire duration of the application.
- **Destruction:** The object is garbage collected when the JVM shuts down. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** This class is completely stateless. It contains no mutable instance fields. All required data is provided as arguments to the deserialize method, and all calculations are performed using local variables.
- **Thread Safety:** The BsonPrefabBufferDeserializer is inherently thread-safe. The deserialize method can be invoked concurrently from multiple threads without risk of data corruption or race conditions, as each invocation operates on its own stack and input data. This makes it suitable for use in parallelized asset loading systems.

## API Surface
The public contract is minimal, consisting of a single deserialization method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(Path, BsonDocument) | PrefabBuffer | O(N) | Deserializes a BsonDocument into a PrefabBuffer. N is the total number of blocks, fluids, and entities. Throws IllegalArgumentException for version mismatches or data constraint violations. Throws IllegalStateException for corrupted data or internal processing failures. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by most game logic developers. It is typically invoked by a higher-level asset management or prefab placement service, which is responsible for reading the BSON file from disk and passing the resulting document to this deserializer.

```java
// Example: Usage within a hypothetical PrefabLoadingService
import com.hypixel.hytale.server.core.prefab.selection.buffer.BsonPrefabBufferDeserializer;
import com.hypixel.hytale.server.core.prefab.selection.buffer.impl.PrefabBuffer;
import org.bson.BsonDocument;
import java.nio.file.Path;

public class PrefabLoadingService {
    public PrefabBuffer loadPrefabFromBson(Path assetPath, BsonDocument prefabData) {
        // The service retrieves the singleton instance
        BsonPrefabBufferDeserializer deserializer = BsonPrefabBufferDeserializer.INSTANCE;

        try {
            // The core deserialization call
            return deserializer.deserialize(assetPath, prefabData);
        } catch (IllegalArgumentException | IllegalStateException e) {
            // CRITICAL: The service must handle exceptions to prevent loading
            // invalid prefabs into the world.
            log.error("Failed to load prefab asset: " + assetPath, e);
            return null;
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Exceptions:** The exceptions thrown by deserialize are not recoverable errors; they indicate a fundamentally invalid asset. These exceptions **must** be caught and handled by the calling system to prevent server instability or world corruption. Continuing execution with a partially loaded or null buffer is unsafe.
- **Direct Instantiation:** This class is a singleton. Do not attempt to create new instances. Always use the static BsonPrefabBufferDeserializer.INSTANCE field.
- **Assuming Forward Compatibility:** The deserializer explicitly throws an exception if the prefab version is newer than what the code supports. Do not suppress this check. Attempting to load a newer prefab format will lead to unpredictable behavior and data loss.

## Data Pipeline
The BsonPrefabBufferDeserializer is a single, critical step in the data pipeline that transforms a persistent asset into a usable runtime object.

> Flow:
> BSON File on Disk -> BSON Parser -> `BsonDocument` -> **BsonPrefabBufferDeserializer** -> `PrefabBuffer` -> World Placement System

