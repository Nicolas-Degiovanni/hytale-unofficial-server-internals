---
description: Architectural reference for ProtocolCodecs
---

# ProtocolCodecs

**Package:** com.hypixel.hytale.server.core.codec
**Type:** Utility

## Definition
```java
// Signature
public final class ProtocolCodecs {
```

## Architecture & Concepts

The ProtocolCodecs class is a static, centralized registry of codec definitions used for data serialization and deserialization throughout the Hytale server. It is not a service or a manager, but rather a foundational utility class that provides the immutable "blueprints" for converting between in-memory Java objects and their serialized representations (e.g., network binary format, JSON/HOCON configuration files).

Its primary architectural role is to enforce **consistency** and **decoupling**. By defining all core data structure codecs in one place, the system guarantees that a Vector3f or a MapMarker is serialized and deserialized identically, whether it is being sent over the network, saved in a world file, or configured in an asset. This decouples the plain Java objects (POJOs) in packages like com.hypixel.hytale.protocol from the specifics of their on-disk or over-the-wire format.

Each public static final field is a fully configured codec instance, typically built using the fluent `BuilderCodec` API. This pattern allows for complex object graphs to be defined declaratively, incorporating nested objects, validation rules, and post-processing hooks. For example, the `ITEM_ANIMATION_CODEC` not only defines the structure of an ItemAnimation but also integrates with the `CommonAssetValidator` to ensure asset paths are valid and hooks into the `BlockyAnimationCache` to prime the cache after decoding.

## Lifecycle & Ownership

-   **Creation:** The codec instances within this class are created as static final fields. They are instantiated and configured once by the JVM's class loader when the ProtocolCodecs class is first referenced. This occurs during the server's initial bootstrap phase, ensuring all codecs are available before any game logic or network communication begins.
-   **Scope:** The codecs have a global, application-wide scope. They persist for the entire lifetime of the server process.
-   **Destruction:** The codecs are garbage collected along with all other static data when the Java Virtual Machine shuts down. There is no manual cleanup or destruction logic.

## Internal State & Concurrency

-   **State:** The ProtocolCodecs class itself is stateless. The `Codec` objects it contains are **effectively immutable**. Their configuration, including field mappings, validators, and metadata, is defined at compile time and finalized during class loading. This configuration does not change during runtime.
-   **Thread Safety:** All codecs defined in this class are **thread-safe**. Because their internal configuration is immutable, they can be safely used by multiple threads simultaneously without locks or synchronization. This is a critical design feature for a high-performance, multi-threaded server environment, allowing network threads, world generation threads, and the main game loop to perform serialization operations concurrently without contention.

## API Surface

The public API consists entirely of static final fields, each representing a specific codec. Below are representative examples of the patterns used.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| VECTOR3F | BuilderCodec<Vector3f> | O(1) | Defines the serialization format for a 3D float vector with keys X, Y, and Z. Used ubiquitously for positions and directions. |
| GAMEMODE | EnumCodec<GameMode> | O(1) | Maps the GameMode enum to its string representation. Includes embedded documentation for keys, which can be consumed by external tooling. |
| MARKER_ARRAY | ArrayCodec<MapMarker> | O(N) | Defines a codec for an array of MapMarker objects, reusing the MARKER codec for each element. N is the number of elements in the array. |
| ITEM_ANIMATION_CODEC | BuilderCodec<ItemAnimation> | O(1) | A complex codec for item animations. Integrates asset validation and a post-decode hook to prime the BlockyAnimationCache. |
| RAIL_CONFIG_CODEC | BuilderCodec<RailConfig> | O(N) | Defines a rail configuration, validating that the number of points is within the required range (2 to 16). |

## Integration Patterns

### Standard Usage

Codecs from this class should always be accessed statically. They are typically passed to higher-level systems like asset loaders or packet serializers, which then invoke the encode and decode methods.

```java
// Example: Using a codec to decode data from a generic source
// (DataSource is a hypothetical interface for network buffers or config files)
DataSource source = getDataSourceFor("player.json");
Vector3f lastKnownPosition = source.readObject("position", ProtocolCodecs.VECTOR3F);

// Example: Reusing a codec to build a more complex one
BuilderCodec<PlayerState> PLAYER_STATE_CODEC = BuilderCodec.builder(PlayerState.class, PlayerState::new)
    .addField(new KeyedCodec<>("Position", ProtocolCodecs.VECTOR3F), ...)
    .addField(new KeyedCodec<>("GameMode", ProtocolCodecs.GAMEMODE), ...)
    .build();
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** The class is final and has no public constructor. It cannot be instantiated with `new ProtocolCodecs()`.
-   **Runtime Modification:** Do not attempt to use reflection to modify the static final codec fields. The entire engine relies on the assumption that these codecs are immutable constants. Modifying them at runtime will lead to unpredictable serialization behavior and system instability.
-   **Redefinition:** Do not define a new, separate codec for a data type that already has a definition in ProtocolCodecs. This fragments the serialization logic and is a guaranteed source of bugs when one definition is updated and the other is not. Always reuse the canonical codecs provided by this class.

## Data Pipeline

ProtocolCodecs does not process data directly; it provides the definitions used by other systems to process data. It sits at the core of any data serialization or deserialization pipeline.

> **Flow (Network Packet Ingress):**
> Raw Byte Buffer -> Packet Deserializer -> **ProtocolCodecs.SOME_PACKET_CODEC.decode()** -> In-Memory Packet Object -> Game Logic

> **Flow (Asset File Loading):**
> JSON/HOCON File on Disk -> Config Parser -> Asset Loader -> **ProtocolCodecs.SOME_ASSET_CODEC.decode()** -> In-Memory Asset Object -> Game Registry

