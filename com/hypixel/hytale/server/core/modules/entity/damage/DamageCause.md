---
description: Architectural reference for DamageCause
---

# DamageCause

**Package:** com.hypixel.hytale.server.core.modules.entity.damage
**Type:** Data Asset

## Definition
```java
// Signature
public class DamageCause implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, DamageCause>> {
```

## Architecture & Concepts

The DamageCause class is a data-driven definition that represents a specific type of damage within the game engine, such as fall damage, fire, or a sword strike. It is not a service or a transient gameplay object; instead, it functions as a configuration asset loaded from the server's file system.

This class is a cornerstone of the Hytale Asset System. Instances are not instantiated directly in game logic via the *new* keyword. Instead, they are deserialized from JSON files at server startup and managed by a central, static **AssetStore**. This design decouples the game's damage mechanics from hardcoded logic, allowing designers and modders to define or alter damage types, their properties, and their behaviors purely through data.

The key architectural features include:

*   **Data-Driven:** All properties, such as whether damage bypasses resistances or causes durability loss, are defined in external JSON asset files.
*   **Central Registry:** The static **ASSET_STORE** provides a single, global point of access for all loaded DamageCause definitions, indexed by a string identifier (e.g., "hytale:fall").
*   **Inheritance:** The *inherits* field enables a hierarchical structure where damage types can be derived from one another, promoting reusability of properties. For example, a "Lava" damage type might inherit from a base "Fire" type.
*   **Network Serialization:** The **toPacket** method provides a clean boundary for converting the comprehensive server-side definition into a lightweight, network-optimized object for client-side effects, such as displaying the correct damage color.

### Lifecycle & Ownership

- **Creation:** Instances are created exclusively by the **AssetStore** during the server's bootstrap phase. The static **CODEC** field, an **AssetBuilderCodec**, defines the deserialization logic that maps JSON key-value pairs to the fields of a DamageCause object.
- **Scope:** Application-scoped. Once loaded, all DamageCause definitions are immutable and persist for the entire lifetime of the server process. They are shared, global constants.
- **Destruction:** Instances are garbage collected only upon server shutdown when the global **AssetRegistry** is cleared. There is no mechanism for unloading or destroying individual DamageCause assets during runtime.

## Internal State & Concurrency

- **State:** Effectively immutable. Although the fields are not declared as final, the system design mandates that they are populated only once during the asset loading phase. Runtime modification of a loaded DamageCause instance is an anti-pattern that would have unpredictable global consequences.
- **Thread Safety:** The class is thread-safe for all read operations. Because the state is treated as immutable post-initialization, multiple threads (e.g., parallel world simulations) can safely access and read from DamageCause objects without locks or synchronization.

    **Warning:** The lazy initialization within the **getAssetStore** method is not inherently thread-safe. This system relies on the assumption that the entire **AssetRegistry** is populated in a single-threaded context during server startup, before any multi-threaded game logic begins.

## API Surface

The public API is designed for read-only access to the damage type's properties.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Returns the central, global registry for all DamageCause assets. |
| getAssetMap() | static IndexedLookupTableAssetMap | O(1) | A convenience method to directly access the map of ID-to-DamageCause. |
| getId() | String | O(1) | Returns the unique string identifier for this damage type (e.g., "hytale:projectile"). |
| getInherits() | String | O(1) | Returns the ID of the parent DamageCause this asset inherits from, if any. |
| isDurabilityLoss() | boolean | O(1) | Determines if this damage should reduce the durability of worn armor or held items. |
| doesBypassResistances() | boolean | O(1) | Determines if this damage should ignore entity resistances and immunities. |
| toPacket() | com.hypixel.hytale.protocol.DamageCause | O(1) | Creates a lightweight network packet representation of this damage cause. |

## Integration Patterns

### Standard Usage

Game logic should never create a DamageCause. The standard pattern is to retrieve a pre-existing, loaded definition from the static asset map using its unique identifier.

```java
// Retrieve the central map of all damage causes
IndexedLookupTableAssetMap<String, DamageCause> damageMap = DamageCause.getAssetMap();

// Fetch a specific damage definition by its ID
DamageCause fallDamage = damageMap.get("hytale:fall");

// Use the definition to drive game logic
if (fallDamage != null && fallDamage.doesBypassResistances()) {
    // Apply damage even if the entity has armor
}
```

### Anti-Patterns (Do NOT do this)

- **Direct Instantiation:** Never use `new DamageCause()`. This bypasses the asset system, creating an incomplete object that is not registered globally. It will not be recognized by other game systems.
- **Runtime Modification:** Do not modify the state of a DamageCause object after it has been loaded. These are shared, global definitions. Modifying one instance will affect all systems that reference it, leading to unpredictable behavior.
- **Access Before Asset Loading:** Calling **getAssetStore** or **getAssetMap** before the server's asset loading phase is complete will result in a NullPointerException or an empty map. All access should occur after the server has fully bootstrapped.

## Data Pipeline

The flow of data from configuration to runtime usage is a one-way pipeline managed by the asset system.

> Flow:
> 1. **JSON Asset File** (e.g., `fall.json`) ->
> 2. **Server Bootstrap** & **AssetRegistry** ->
> 3. **DamageCause.CODEC** deserializes the file ->
> 4. An instance of **DamageCause** is created and stored in the static **AssetStore** ->
> 5. **Game Logic** (e.g., Physics System) retrieves the instance by ID ->
> 6. **Combat System** uses its properties to calculate effects ->
> 7. **Network Layer** calls **toPacket()** to notify the client.

