---
description: Architectural reference for UnarmedInteractions
---

# UnarmedInteractions

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction
**Type:** Data Asset

## Definition
```java
// Signature
public class UnarmedInteractions implements JsonAssetWithMap<String, DefaultAssetMap<String, UnarmedInteractions>> {
```

## Architecture & Concepts
The UnarmedInteractions class is a data-driven configuration asset, not a service or manager. Its sole purpose is to define the set of default behaviors that occur when a player performs an action without an item equipped in their active hand. It acts as a routing table, mapping a specific player input, represented by the **InteractionType** enum, to a more complex interaction definition identified by a string key.

This class is a fundamental component of the server's interaction module. It decouples the low-level input signals (e.g., left-click, swap item) from the high-level game logic that executes. By defining these mappings in data files (JSON), designers can alter unarmed behaviors without changing core engine code.

The system relies on a static **CODEC** field, which uses the **AssetBuilderCodec** to define the deserialization process from JSON. This codec is responsible for constructing instances, inheriting properties from parent assets, and performing post-load validation and finalization, such as making the internal collections immutable.

## Lifecycle & Ownership
- **Creation:** Instances of UnarmedInteractions are never created directly via a constructor in game logic. They are exclusively instantiated by the **AssetRegistry** during the server's asset loading phase. The static **CODEC** is invoked by the asset system to parse a corresponding JSON file and construct the object in memory.

- **Scope:** An UnarmedInteractions object is a global, session-scoped singleton for its specific configuration. Once loaded, it persists immutably in the static **ASSET_MAP** for the entire duration of the server's runtime.

- **Destruction:** These objects are not explicitly destroyed. They are garbage collected along with all other assets when the server shuts down and the Java Virtual Machine terminates.

## Internal State & Concurrency
- **State:** The internal state of a loaded UnarmedInteractions instance is **immutable**. The primary state, the **interactions** map, is wrapped in an unmodifiable view via `Collections.unmodifiableMap` within the codec's **afterDecode** hook. This guarantees that the configuration cannot be altered at runtime.

- **Thread Safety:** This class is inherently thread-safe. Its immutability ensures that it can be safely read from any thread without locks or synchronization. The only mutable element is the static **ASSET_MAP**, whose lazy initialization is managed by the thread-safe **AssetRegistry** system.

## API Surface
The public API is minimal, designed for read-only access to the configuration data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetMap() | static DefaultAssetMap | O(1) | Retrieves the global, lazily-initialized map of all loaded UnarmedInteractions assets. |
| getId() | String | O(1) | Returns the unique identifier for this specific unarmed configuration, e.g., "Empty". |
| getInteractions() | Map | O(1) | Returns an unmodifiable map linking an InteractionType to an interaction definition ID. |

## Integration Patterns

### Standard Usage
The correct pattern is to retrieve a specific configuration, typically the default "Empty" one, from the static asset map and then query it for the desired interaction.

```java
// How a developer should normally use this
// Retrieve the global map of all unarmed interaction configurations
DefaultAssetMap<String, UnarmedInteractions> unarmedMap = UnarmedInteractions.getAssetMap();

// Get the default configuration for a completely empty hand
UnarmedInteractions defaultInteractions = unarmedMap.get(UnarmedInteractions.DEFAULT_UNARMED_ID);

// Find the ID of the interaction to run for a primary attack (left-click)
String interactionId = defaultInteractions.getInteractions().get(InteractionType.Primary);

// The interactionId is then used to look up and execute the full interaction logic
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new UnarmedInteractions()`. This will create a useless, un-initialized object that is not registered with the asset system and will cause NullPointerExceptions.

- **Runtime Modification:** Do not attempt to cast or use reflection to modify the map returned by **getInteractions**. It is intentionally immutable and any such attempt will either fail or lead to undefined behavior.

- **Premature Access:** Do not call **getAssetMap** before the server's main asset loading phase is complete. Doing so may return a null or incomplete map, leading to cascading failures in dependent systems.

## Data Pipeline
UnarmedInteractions is a source of configuration data, loaded from disk and consumed by the game's interaction logic. It does not process data itself.

> Flow:
> `unarmed.json` on Disk -> Server Asset Loader -> **UnarmedInteractions.CODEC** -> **UnarmedInteractions Instance** -> In-Memory **ASSET_MAP** -> Interaction Handling Service -> Game Logic Execution

