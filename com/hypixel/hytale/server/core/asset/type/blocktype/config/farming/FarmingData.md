---
description: Architectural reference for FarmingData
---

# FarmingData

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config.farming
**Type:** Data Model / Configuration Object

## Definition
```java
// Signature
public class FarmingData {
```

## Architecture & Concepts
The FarmingData class is a passive data model that encapsulates all configuration related to the growth and harvesting mechanics of a farmable block. It is not a service or an active component; instead, it serves as a structured, in-memory representation of a farming configuration defined in an external asset file, typically JSON.

Its primary role is to be deserialized and attached to a parent BlockType asset during the server's asset loading phase. The entire lifecycle and behavior of a crop—from its initial planting stage, through its growth cycle, to its final harvestable state—is defined by an instance of this class.

The static CODEC field is the cornerstone of this class's design. It leverages the engine's powerful codec system to map raw asset data into a strongly-typed Java object, including complex nested structures like growth stages and soil requirements. This declarative approach separates game data from game logic, allowing designers to configure farming behavior without modifying engine code.

## Lifecycle & Ownership
- **Creation:** FarmingData instances are created exclusively by the Hytale codec system during asset deserialization. The static `CODEC` field of type BuilderCodec dictates the entire construction process. The `afterDecode` hook provides critical post-deserialization validation, ensuring internal consistency (e.g., that the `startingStageSet` actually exists within the defined `stages`).

- **Scope:** The lifetime of a FarmingData object is strictly bound to its parent BlockType asset. It is loaded into memory when the BlockType is loaded by the AssetManager and persists as long as the BlockType is part of the active asset registry.

- **Destruction:** The object is eligible for garbage collection when its parent BlockType is unloaded, typically during a server shutdown or a world change that purges the associated asset set.

## Internal State & Concurrency
- **State:** The object's state is mutable only during the brief window of its creation by the BuilderCodec. Once deserialization is complete and the object is "published" as part of a BlockType asset, it should be treated as **effectively immutable**. All fields are populated once, and the public API provides no methods for mutation.

- **Thread Safety:** The class is inherently thread-safe for read operations. Because its state does not change after initialization, game logic systems (such as the world tick or block update threads) can safely access its configuration data concurrently without requiring locks or other synchronization primitives.

## API Surface
The public API consists exclusively of read-only accessors for its internal configuration state (e.g., getStages, getSoilConfig). There are no methods for mutation. The primary consumer of this API is the server's core farming simulation logic, which reads this data to determine how a block should behave over time.

## Integration Patterns

### Standard Usage
A developer should never instantiate or manage a FarmingData object directly. It is always accessed as a component of an already-loaded BlockType asset.

```java
// Correctly retrieve the farming configuration from a BlockType
BlockType cropBlock = blockTypeRegistry.get("hytale:wheat");
FarmingData farmingConfig = cropBlock.getFarmingData();

if (farmingConfig != null) {
    // Use the configuration to inform game logic
    String startingStage = farmingConfig.getStartingStageSet();
    // ...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new FarmingData()`. This will result in an uninitialized, invalid object that bypasses all codec-level validation. The game engine will fail to process such an object correctly, likely resulting in a NullPointerException or an IllegalStateException.

- **Runtime Modification:** Although the fields are not declared as final, modifying the state of a FarmingData object after it has been loaded is a severe anti-pattern. This would lead to unpredictable and inconsistent farming behavior across the server, as all instances of that block type share the same configuration object.

## Data Pipeline
FarmingData is the result of a data transformation pipeline that begins with a raw asset file on disk and ends with a usable configuration object in the game engine.

> Flow:
> Asset File (e.g., wheat.json) -> AssetManager -> **FarmingData.CODEC** -> **FarmingData Instance** -> Attached to BlockType Asset -> Read by Farming System

---

# FarmingData.SoilConfig

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config.farming
**Type:** Data Model / Configuration Object

## Definition
```java
// Signature
public static class SoilConfig {
```

## Architecture & Concepts
SoilConfig is a nested data model within FarmingData, specifically designed to define the relationship between a farmable block and the block it must be planted on (the "soil"). It specifies which block is a valid target for planting and defines the "lifetime" of that soil, which can be used for mechanics like tilled earth reverting to dirt over time.

Like its parent, it is a passive data container instantiated by the codec system. Its existence is entirely dependent on being defined within a FarmingData configuration.

## Lifecycle & Ownership
- **Creation:** Instantiated by the parent FarmingData.CODEC when a "SoilConfig" key is present in the asset data. Its own nested BuilderCodec handles its construction and validation, including a late-binding validator to ensure the `targetBlock` string corresponds to a real, loaded BlockType.

- **Scope:** The lifecycle of a SoilConfig object is identical to and completely dependent on its parent FarmingData instance. It cannot exist independently.

- **Destruction:** It is garbage collected along with its parent FarmingData object.

## Internal State & Concurrency
- **State:** Effectively immutable after creation via the codec.
- **Thread Safety:** Inherently thread-safe for all read operations, mirroring the guarantees of its parent class.

## Integration Patterns

### Standard Usage
SoilConfig is accessed through an instance of FarmingData. It is used by game logic to validate planting actions and to manage the state of the soil block itself.

```java
// Check if a block can be planted on a target soil
FarmingData farmingConfig = crop.getFarmingData();
SoilConfig soilConfig = farmingConfig.getSoilConfig();

if (soilConfig != null && soilConfig.getTargetBlock().equals(soilBlock.getName())) {
    // Logic for successful planting
}
```

