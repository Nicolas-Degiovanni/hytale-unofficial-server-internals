---
description: Architectural reference for BenchTierLevel
---

# BenchTierLevel

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config.bench
**Type:** Transient

## Definition
```java
// Signature
public class BenchTierLevel implements NetworkSerializable<com.hypixel.hytale.protocol.BenchTierLevel> {
```

## Architecture & Concepts
The BenchTierLevel class is a server-side configuration model that represents the properties of a single upgrade level for a crafting bench. It is not a service or a manager, but rather a pure Data Transfer Object (DTO) whose structure is defined by game designers in external asset files, typically JSON.

Its primary architectural role is to act as a strongly-typed, in-memory representation of data that is deserialized from configuration files. The static final field **CODEC** is the cornerstone of this design. It integrates the class with Hytale's reflection-based asset loading system, allowing the engine to automatically parse bench configuration data and instantiate BenchTierLevel objects.

Furthermore, by implementing the NetworkSerializable interface, this class serves as a critical bridge between the server's internal asset configuration and the client-facing network protocol. The toPacket method facilitates the translation of this server-side data model into a network-ready packet format for transmission to clients.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale **CODEC** system during server asset loading. The engine's AssetManager reads a block definition file, identifies the bench tier data, and invokes the static BenchTierLevel.CODEC to deserialize the data into a new BenchTierLevel object. Manual instantiation is a design violation.

- **Scope:** The lifetime of a BenchTierLevel instance is bound to the parent configuration object that owns it, such as a BenchComponent or BlockType. It is loaded once at server startup and persists in memory for the entire server session.

- **Destruction:** The object is marked for garbage collection when its parent asset is unloaded, typically during a server shutdown or a full asset reload. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** The state is **effectively immutable**. Although its fields are not declared final, the class is designed to be instantiated once from a static data source and then only read from. Modifying its state at runtime would lead to a divergence between the in-memory state and the source asset files, causing unpredictable behavior.

- **Thread Safety:** This class is **conditionally thread-safe**. As a simple data container, it is safe for concurrent reads from multiple threads. However, it is not inherently thread-safe for writes. This is not a practical concern, as all instantiation and population occurs synchronously on the main server thread during the asset loading phase. Subsequent access from game logic threads is read-only.

## API Surface
The public API is limited to data access and network serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getCraftingTimeReductionModifier() | float | O(1) | Returns the crafting speed multiplier for this tier. |
| getUpgradeRequirement() | BenchUpgradeRequirement | O(1) | Returns the requirements needed to upgrade to this tier. |
| getExtraInputSlot() | int | O(1) | Returns the number of additional input slots this tier provides. |
| getExtraOutputSlot() | int | O(1) | Returns the number of additional output slots this tier provides. |
| toPacket() | com.hypixel.hytale.protocol.BenchTierLevel | O(1) | Converts this server-side model into its network protocol equivalent. |

## Integration Patterns

### Standard Usage
Developers should never create a BenchTierLevel directly. Instead, they should retrieve it from a higher-level configuration object that has been loaded via the asset system.

```java
// Correctly retrieve a BenchTierLevel from a parent configuration component
// which is itself retrieved from a loaded BlockType asset.

BlockType craftingBenchAsset = assetManager.get(BlockType.class, "my_mod:advanced_bench");
BenchComponent benchConfig = craftingBenchAsset.getComponent(BenchComponent.class);

// Access a specific tier's data, which is a BenchTierLevel instance
BenchTierLevel tierTwoData = benchConfig.getTier(2);

if (tierTwoData != null) {
    float speedBonus = tierTwoData.getCraftingTimeReductionModifier();
    // Use the configuration data in game logic...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BenchTierLevel()`. This bypasses the asset loading pipeline and creates a "phantom" configuration object that is disconnected from the server's authoritative state. All configuration must originate from asset files.

- **Runtime State Mutation:** Do not attempt to modify the fields of a BenchTierLevel instance after it has been loaded. This creates an inconsistent state that will not be persisted and will be reverted on the next server restart.

## Data Pipeline
The BenchTierLevel class exists at two key points in the server's data flow: during initial asset loading and during network synchronization.

> **Asset Loading Flow:**
> JSON Asset File -> AssetManager -> **BenchTierLevel.CODEC** -> **BenchTierLevel Instance** (In-Memory)

> **Network Serialization Flow:**
> Game Logic requests data -> **BenchTierLevel Instance** -> toPacket() -> `protocol.BenchTierLevel` -> Network Encoder -> TCP Packet -> Client

