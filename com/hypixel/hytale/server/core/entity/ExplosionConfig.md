---
description: Architectural reference for ExplosionConfig
---

# ExplosionConfig

**Package:** com.hypixel.hytale.server.core.entity
**Type:** Configuration Model

## Definition
```java
// Signature
public class ExplosionConfig {
```

## Architecture & Concepts
The ExplosionConfig class is a passive data structure that defines the physical properties and gameplay effects of an explosion event on the server. It embodies a data-driven design philosophy, allowing game designers to define and tune explosion behaviors in external asset files (e.g., JSON) without modifying the core engine code.

This class is not an active component; it does not contain logic. Instead, it serves as a parameter object for the server's explosion simulation systems. Its primary architectural feature is the static **CODEC** field, which integrates it directly into Hytale's asset serialization framework. This codec is responsible for translating raw asset data into a fully-formed ExplosionConfig Java object.

The configuration supports inheritance via the `appendInherited` directive in its codec definition. This allows for the creation of a base explosion template from which more specific explosion types can derive, overriding only the necessary properties. For example, a "Grenade" explosion might inherit from a "Default" explosion but specify a smaller radius and different damage values.

## Lifecycle & Ownership
- **Creation:** Instances of ExplosionConfig are almost exclusively created by the Hytale serialization engine. When game logic requires an explosion (e.g., a TNT block detonates), the system requests the corresponding configuration asset. The `AssetManager` locates the asset file and uses the static `ExplosionConfig.CODEC` to deserialize the data into a new object instance.
- **Scope:** The lifetime of an ExplosionConfig object is typically transient and bound to a single explosion event. It is loaded, passed to the simulation logic, and then becomes eligible for garbage collection once the event processing is complete.
- **Destruction:** Managed entirely by the Java garbage collector. There are no manual cleanup or disposal methods.

## Internal State & Concurrency
- **State:** The class is a mutable container for explosion properties. It is populated with default values upon construction, which are then overwritten by the deserialization process based on the contents of the asset file. Once an instance is passed to the simulation logic, it should be treated as *effectively immutable* for the duration of that operation.
- **Thread Safety:** This class is **not thread-safe**. It contains no internal synchronization mechanisms. It is designed to be created, populated, and read within a single thread, typically the main server game loop or a dedicated world-processing thread. Concurrently modifying an instance from multiple threads will lead to undefined behavior.

## API Surface
The primary public contract is not a method, but the static codec used for data binding. The fields themselves are protected and intended for access by the simulation engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | BuilderCodec | O(N) | The static serializer and deserializer. This is the sole entry point for creating instances from game assets. N is the number of properties in the asset file. |

## Integration Patterns

### Standard Usage
A developer typically does not interact with this class directly. Instead, they trigger a world event, and the engine handles the loading and application of the configuration. The system-level interaction would resemble the following:

```java
// PSEUDOCODE: How the engine uses ExplosionConfig
// A developer would call something simpler, like world.createExplosion(...)

// 1. An asset key for the desired explosion type is retrieved.
AssetKey<ExplosionConfig> tntExplosionKey = AssetKeys.get("hytale:tnt_explosion");

// 2. The AssetManager loads and deserializes the config using the CODEC.
ExplosionConfig config = assetManager.load(tntExplosionKey);

// 3. The config object is passed to the simulation system.
ExplosionSimulator.processExplosion(world, position, config);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ExplosionConfig()`. This bypasses the data-driven asset pipeline and results in an object with only default values, ignoring designer-specified configurations. All instances must be created via the asset system.
- **State Mutation After Load:** Modifying the fields of an ExplosionConfig object after it has been loaded and passed to a system is dangerous. If configurations are ever cached for performance, such mutations could have widespread, unpredictable effects on all subsequent explosions of that type.

## Data Pipeline
The ExplosionConfig class is a key component in the data flow from game assets to world simulation.

> Flow:
> Game Asset File (JSON) -> AssetManager -> **ExplosionConfig.CODEC** -> In-Memory ExplosionConfig Instance -> Explosion Simulation Logic -> World State Change (Damaged Blocks & Entities)

