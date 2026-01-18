---
description: Architectural reference for SpawnSuppressorEntry
---

# SpawnSuppressorEntry

**Package:** com.hypixel.hytale.server.spawning.suppression
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class SpawnSuppressorEntry {
```

## Architecture & Concepts
The SpawnSuppressorEntry class is a fundamental data structure, not a service. It functions as a self-contained record that represents a single point of spawn suppression within the game world. Each instance defines a specific coordinate and an identifier that links it to a particular suppression rule or source (e.g., a player-placed structure, a quest area).

The most critical architectural feature of this class is the public static **CODEC** field. This exposes a Hytale BuilderCodec that defines the serialization and deserialization contract for the object. This pattern allows SpawnSuppressorEntry to be seamlessly loaded from world data files, zone configurations, or network packets without coupling the spawning system directly to the data format. It effectively acts as a data schema, ensuring that persisted suppression data can be reliably hydrated into a live object.

This class is a leaf node in the spawning system's data model. It holds state but contains no logic, delegating all processing to higher-level managers like a hypothetical SpawnSuppressionManager.

## Lifecycle & Ownership
- **Creation:** Instances are primarily created by the Hytale serialization framework when loading world data. The private no-argument constructor exists solely to support this deserialization process via the defined CODEC. Manual instantiation using the public constructor is reserved for programmatic, runtime-based spawn suppression.

- **Scope:** The lifetime of a SpawnSuppressorEntry instance is strictly tied to its container, typically a collection within a world or zone management service. It persists as long as the suppression rule is active in the game world.

- **Destruction:** Instances are marked for garbage collection when the rule they represent is removed or when the world chunk or zone they belong to is unloaded. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** The internal state consists of a String identifier and a Vector3d position. The object is technically mutable, as it lacks final fields. However, it is designed to be treated as an **immutable value object** after its initial creation or deserialization. Modifying its state post-creation can lead to desynchronization with the core spawning system.

- **Thread Safety:** This class is **not thread-safe**. It is a plain data container with no internal locking mechanisms. All access and mutation must be externally synchronized, typically by confining its use to the main server thread or a system that guarantees thread-safe access to its collections.

## API Surface
The public contract is minimal, focusing on data access and serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | BuilderCodec | O(1) | **Critical.** A static definition for serializing and deserializing instances of this class. |
| getPosition() | Vector3d | O(1) | Returns the world-space coordinate of the suppression point. |
| getSuppressionId() | String | O(1) | Returns the unique identifier for the suppression rule. |

## Integration Patterns

### Standard Usage
Direct interaction with this class is uncommon. It is typically loaded and managed by a higher-level system. The primary integration point is the static CODEC field, used by data loaders.

```java
// Example of programmatic creation for a runtime effect
// NOTE: This is a secondary use case. Most entries are loaded from data.

Vector3d location = new Vector3d(100, 64, -250);
String reason = "world_event_boss_arena";

SpawnSuppressorEntry newSuppressor = new SpawnSuppressorEntry(reason, location);

// The entry would then be registered with the appropriate manager
spawnSuppressionManager.addEntry(newSuppressor);
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not modify the object's state after it has been registered with a system. This will not trigger any updates and can lead to inconsistent behavior.
- **Cross-Thread Sharing:** Do not pass an instance to another thread without ensuring external synchronization. The internal fields are not volatile or protected by locks.
- **Ignoring the CODEC:** Do not attempt to implement custom serialization logic for this class. The provided CODEC is the canonical method for persistence and must be used to ensure forward compatibility.

## Data Pipeline
This class represents a data record at the beginning of a decision pipeline. It does not process data itself.

> Flow:
> World Data File (JSON/Binary) -> Hytale Codec Framework -> **SpawnSuppressorEntry Instance** -> SpawnSuppressionManager Collection -> Spawning Algorithm Logic Check

