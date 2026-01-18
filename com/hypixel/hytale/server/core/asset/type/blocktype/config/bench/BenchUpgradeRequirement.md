---
description: Architectural reference for BenchUpgradeRequirement
---

# BenchUpgradeRequirement

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config.bench
**Type:** Configuration Model / DTO

## Definition
```java
// Signature
public class BenchUpgradeRequirement implements NetworkSerializable<com.hypixel.hytale.protocol.BenchUpgradeRequirement> {
```

## Architecture & Concepts
The BenchUpgradeRequirement class is a data-centric model that represents the specific conditions—materials and time—required to perform an upgrade on a crafting bench or a similar interactive block. This class is not a service or manager; it is a passive data container whose structure is defined by and loaded from external game asset files, likely in a format like JSON or HOCON.

Its primary architectural significance lies in the static **CODEC** field. This `BuilderCodec` instance provides a declarative contract for deserialization. It dictates how raw asset data is mapped to this Java object, including key names ("Material", "TimeSeconds"), data types, and validation rules (e.g., time must be non-negative). This pattern decouples the game's configuration data from the engine's loading mechanisms, allowing for robust and maintainable asset pipelines.

Furthermore, by implementing the NetworkSerializable interface, this class serves as a critical bridge between the server's internal configuration state and the client-facing network protocol. The `toPacket` method translates this server-side model into a network-ready format, ensuring that clients receive the correct upgrade information.

## Lifecycle & Ownership
- **Creation:** Instances of BenchUpgradeRequirement are not intended for manual instantiation by game logic. They are created exclusively by the server's asset loading system during the bootstrap phase. The static CODEC field is invoked by the asset pipeline to deserialize the configuration and construct the object.
- **Scope:** The object's lifetime is bound to its parent configuration asset, such as a `BenchType` definition. Once loaded, it persists in memory for the entire server session as part of the immutable asset registry.
- **Destruction:** The object is marked for garbage collection when the server shuts down and the asset registries are cleared. There is no manual destruction mechanism.

## Internal State & Concurrency
- **State:** This object is **effectively immutable** after its creation during asset loading. While the `input` array is technically mutable, it is a severe design violation to modify it at runtime. The fields are populated once by the CODEC and are not exposed for modification.
- **Thread Safety:** The class is **inherently thread-safe** for read operations. Its immutable nature guarantees that multiple game logic threads (e.g., handling different players interacting with benches) can access its data concurrently without requiring locks or synchronization primitives.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getInput() | MaterialQuantity[] | O(1) | Returns the array of materials required for the upgrade. **Warning:** The returned array must be treated as read-only. |
| getTimeSeconds() | float | O(1) | Returns the time in seconds required for the upgrade to complete. |
| toPacket() | com.hypixel.hytale.protocol.BenchUpgradeRequirement | O(N) | Translates this object into its network protocol equivalent. N is the number of materials in the input array. |

## Integration Patterns

### Standard Usage
A developer should never create an instance of this class directly. Instead, it should be retrieved from a higher-level configuration object that has been loaded from the asset registry.

```java
// Correctly retrieve the requirement from a parent configuration object
BenchType benchConfig = assetRegistry.get(BenchType.class, "hytale:workbench");
BenchUpgradeConfiguration upgrade = benchConfig.getUpgradeConfiguration();

// Access the data for use in game logic
BenchUpgradeRequirement requirement = upgrade.getRequirementForNextLevel();
MaterialQuantity[] materials = requirement.getInput();
float time = requirement.getTimeSeconds();

// Use the retrieved data to validate a player's upgrade attempt
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new BenchUpgradeRequirement()`. This bypasses the canonical asset pipeline and creates disconnected, non-standard game data that will not be recognized by other systems.
- **State Mutation:** Do not modify the contents of the array returned by `getInput()`. This array is a shared, global configuration state. Modifying it will cause unpredictable behavior for all other game logic referencing the same asset.

```java
// ANTI-PATTERN: Modifying shared state
BenchUpgradeRequirement requirement = getRequirementFromAsset();
MaterialQuantity[] materials = requirement.getInput();
materials[0] = null; // DANGEROUS: This corrupts the server's global asset configuration.
```

## Data Pipeline
This class primarily functions as a container within the server's data flow, transforming static configuration data into network-transmissible packets.

> Flow:
> Game Asset File (e.g., JSON) -> Server Asset Deserializer (using CODEC) -> **BenchUpgradeRequirement** (In-Memory Object) -> Game Logic -> `toPacket()` -> Network Packet -> Client Rendering

