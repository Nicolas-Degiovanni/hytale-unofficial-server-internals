---
description: Architectural reference for RespawnConfig
---

# RespawnConfig

**Package:** com.hypixel.hytale.server.core.asset.type.gameplay
**Type:** Transient Data Model

## Definition
```java
// Signature
public class RespawnConfig {
```

## Architecture & Concepts
The RespawnConfig class is a server-side data model that encapsulates configuration parameters for player respawn behavior. It is not an active service or manager; rather, it is a passive data structure whose sole purpose is to hold values deserialized from a configuration asset file.

The primary architectural feature of this class is the static **CODEC** field. This `BuilderCodec` instance defines the contract for how a RespawnConfig object is constructed from a keyed data source, such as a JSON or HOCON file. It specifies the expected keys (e.g., RadiusLimitRespawnPoint), data types, and validation rules (e.g., values must be greater than 0). This pattern decouples the game logic from the specifics of the configuration file format, allowing asset definitions to evolve independently.

This class is a fundamental component of the server's data-driven design, enabling world creators and developers to tune gameplay mechanics without modifying engine source code.

### Lifecycle & Ownership
- **Creation:** Instances of RespawnConfig are created exclusively by the Hytale asset loading and codec framework. The static CODEC field is used by a higher-level asset manager to deserialize a configuration file into a new RespawnConfig object. The `BuilderCodec` invokes the default constructor and populates the fields based on the asset data.
- **Scope:** The lifetime of a RespawnConfig instance is typically tied to the scope of the world or game session it was loaded for. It is not a global singleton and multiple instances could exist if, for example, different game modes load different configurations.
- **Destruction:** The object is marked for garbage collection when the server or world that owns the configuration is shut down and all references to the instance are released.

## Internal State & Concurrency
- **State:** The state of RespawnConfig is mutable during the deserialization process managed by its CODEC. After population, it is intended to be treated as a read-only, immutable snapshot of the configuration. It contains no logic, only data fields representing respawn rules.
- **Thread Safety:** **This class is not thread-safe.** It is a simple data container without any internal locking or synchronization. It is designed to be instantiated and populated on a dedicated loading thread and subsequently read by the main server thread. Unsynchronized, concurrent access, especially writes, will result in data races and undefined server behavior.

## API Surface
The public contract is primarily defined by the static CODEC, which governs serialization, not by instance methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | BuilderCodec | N/A | Static codec defining the asset loading contract, including keys, types, and validation rules. This is the primary entry point for the asset system. |

## Integration Patterns

### Standard Usage
A RespawnConfig object should never be instantiated directly. It is always loaded by the server's asset management system using its public CODEC. Game systems then retrieve the loaded instance to query its values.

```java
// Conceptual example of a server system loading and using the config
// The actual API may differ.
RespawnConfig config = assetManager.load("gamedata/respawn.json", RespawnConfig.CODEC);

int radiusLimit = config.getRadiusLimitRespawnPoint();
// ... use radiusLimit in player respawn logic
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new RespawnConfig()`. This bypasses the asset loading pipeline and results in an object with hardcoded default values, ignoring the server's actual configuration. This can lead to subtle and difficult-to-diagnose bugs in gameplay behavior.
- **Runtime Modification:** Do not modify the fields of a RespawnConfig instance after it has been loaded and shared with other systems. It is designed as a read-only data source. Modifying its state at runtime can create an inconsistent view of the server's configuration, leading to unpredictable behavior.

## Data Pipeline
The flow of data for this component is unidirectional, from a physical file to an in-memory object used by game logic.

> Flow:
> Configuration Asset (File) -> Server Asset Loader -> **RespawnConfig.CODEC** -> **RespawnConfig Instance** -> Player Respawn System

