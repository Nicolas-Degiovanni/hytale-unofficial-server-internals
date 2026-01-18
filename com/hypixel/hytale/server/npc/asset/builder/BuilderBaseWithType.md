---
description: Architectural reference for BuilderBaseWithType
---

# BuilderBaseWithType

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Framework Component / Abstract Builder

## Definition
```java
// Signature
public abstract class BuilderBaseWithType<T> extends BuilderBase<T> implements ISpawnable {
```

## Architecture & Concepts
BuilderBaseWithType is an abstract base class that forms a foundational component of the server's asset-to-entity pipeline, specifically for Non-Player Characters (NPCs). Its primary architectural role is to enforce and standardize the presence of a `type` discriminator within NPC asset configuration files.

This class acts as a contract, ensuring that any asset configuration processed by one of its subclasses contains a valid, non-empty type identifier. This identifier is crucial for the subsequent factory or spawning systems to determine the concrete class of the NPC to instantiate. For example, a JSON file with `"Type": "hytale:goblin"` would be processed by a builder that stores `hytale:goblin`, allowing the spawning system to create a Goblin entity.

By extending BuilderBase, it inherits a suite of low-level JSON parsing and validation utilities. The implementation of the ISpawnable interface is a marker that signifies that the ultimate product of this builder, the object of type T, is intended to be an entity that can be spawned into the game world.

## Lifecycle & Ownership
- **Creation:** As an abstract class, BuilderBaseWithType is never instantiated directly. Concrete subclasses (e.g., `HostileNpcBuilder`, `AmbientCreatureBuilder`) are instantiated transiently by the asset loading system when a corresponding JSON configuration file is discovered and parsed.

- **Scope:** The lifetime of a subclass instance is extremely short. It exists only for the duration of parsing a single JSON asset. It serves as a temporary state accumulator.

- **Destruction:** The object is eligible for garbage collection immediately after the final entity configuration (the object of type T) is constructed and returned from its `build` method (inherited from BuilderBase). It holds no references and is not intended to persist.

## Internal State & Concurrency
- **State:** The class maintains a single piece of mutable state: the `type` string. This field is null upon creation and is populated by a call to `readTypeKey` during the JSON parsing phase. The entire object is a temporary, mutable container by design.

- **Thread Safety:** **Not thread-safe.** Builder instances are designed for synchronous, single-threaded use within the asset loading pipeline. Sharing a builder instance across multiple threads will result in severe race conditions and corrupted state. The asset loading system must guarantee that each file is processed by a new, dedicated builder instance.

## API Surface
The public contract is focused on configuration and data retrieval during the build process.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readCommonConfig(JsonElement) | Builder<T> | O(1) | Overrides parent method. Currently a passthrough to the base implementation. |
| readTypeKey(JsonElement, String) | void | O(1) | Parses the specified key from the JSON, validates it is a non-empty string, and sets the internal `type` field. |
| readTypeKey(JsonElement) | void | O(1) | Convenience overload that assumes the JSON key is "Type". |
| getType() | String | O(1) | Returns the `type` string that was parsed from the configuration. Returns null if `readTypeKey` has not been called. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. Instead, they create concrete implementations that define a specific type of NPC. The system's asset loader then uses this implementation.

```java
// 1. A concrete subclass is defined for a specific NPC category.
public class VillagerBuilder extends BuilderBaseWithType<VillagerData> {

    @Override
    public VillagerBuilder read(JsonElement data) {
        // The system calls read, which in turn calls the base methods
        readCommonConfig(data);
        readTypeKey(data, "villagerType"); // Use a custom key
        // ... read other villager-specific properties
        return this;
    }

    @Override
    public VillagerData build() {
        // Use the parsed type to construct the final data object
        String villagerProfession = getType();
        return new VillagerData(villagerProfession, ...);
    }
}

// 2. The asset system uses the builder (conceptual)
VillagerBuilder builder = new VillagerBuilder();
builder.read(jsonFromFile);
VillagerData finalData = builder.build();
```

### Anti-Patterns (Do NOT do this)
- **State Re-use:** Do not attempt to reuse a builder instance to parse a second asset file. The internal state will be corrupted. Always create a new instance for each asset.
- **Manual Instantiation:** While subclasses can be instantiated with `new`, they should only be created by the asset loading framework to ensure the correct lifecycle and context.
- **Calling `getType` Before `readTypeKey`:** Accessing the type before the JSON has been parsed will result in a null value, likely leading to a NullPointerException downstream.

## Data Pipeline
This class is a critical transformation step in the server's data loading pipeline, converting raw configuration data into structured, type-safe state.

> Flow:
> JSON Asset on Disk -> AssetLoader Service -> Gson Parser (produces JsonElement) -> **Concrete Subclass of BuilderBaseWithType** -> Final `build()` call -> Spawnable NPC Data Object

