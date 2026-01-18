---
description: Architectural reference for FogOptions
---

# FogOptions

**Package:** com.hypixel.hytale.server.core.asset.type.weather.config
**Type:** Configuration Model / Transient

## Definition
```java
// Signature
public class FogOptions {
```

## Architecture & Concepts
The FogOptions class is a server-side data model that encapsulates advanced fog rendering parameters. It is a passive data structure, not an active engine component. Its primary role is to serve as a deserialization target for weather or zone configuration assets.

This class acts as a bridge between static configuration files on the server and the client-side rendering engine. The server loads a weather profile from an asset file, which is decoded into a FogOptions instance. This instance is then used to construct a network packet, which is sent to the client. This mechanism allows server administrators and content creators to define and enforce specific atmospheric and visual styles for different game worlds or zones, overriding the client's default rendering behavior.

The static CODEC field is the cornerstone of this class, defining a declarative schema for parsing the configuration and mapping it to the object's fields. This approach centralizes the serialization logic and separates it from the data-holding responsibilities of the class itself.

## Lifecycle & Ownership
- **Creation:** An instance of FogOptions is created exclusively by the Hytale Codec system during the asset loading phase. The static BuilderCodec field acts as the factory, parsing a configuration file (e.g., a weather asset) and populating a new instance. Direct instantiation is a critical anti-pattern.
- **Scope:** The lifetime of a FogOptions object is bound to its parent configuration asset, such as a WeatherProfile. It persists in memory on the server as long as that weather profile is active or cached.
- **Destruction:** The object is marked for garbage collection when its owning configuration is unloaded or replaced. It has no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** The state is mutable upon construction. However, by design, it should be considered **effectively immutable** after being deserialized from a configuration asset. It is a pure data container whose purpose is to hold static, pre-defined values.
- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization primitives. It is safe for concurrent reads *only after* it has been fully constructed and populated by the single-threaded asset loading pipeline.

**WARNING:** Modifying a FogOptions instance at runtime after its initial creation is a severe anti-pattern and will lead to unpredictable rendering behavior and state desynchronization between server configuration and connected clients.

## API Surface
The public API is composed almost entirely of accessors and a single transformation method for network serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isIgnoreFogLimits() | boolean | O(1) | Returns true if client-side fog distance limits should be bypassed. |
| getEffectiveViewDistanceMultiplier() | float | O(1) | Returns the multiplier for the maximum fog distance, based on render distance. |
| getFogHeightCameraFixed() | Float | O(1) | Returns a fixed, absolute value for height-based fog, or null if disabled. |
| getFogHeightCameraOffset() | float | O(1) | Returns the vertical offset applied to the camera's position for fog calculations. |
| toPacket() | com.hypixel.hytale.protocol.FogOptions | O(1) | **Primary Action.** Transforms this configuration model into a network-serializable protocol message for client synchronization. |

## Integration Patterns

### Standard Usage
The intended use is for an engine service, like a WeatherService, to retrieve the fully-formed FogOptions object from a loaded asset and use it to update clients.

```java
// A server-side system obtains the weather profile for a zone
WeatherProfile activeWeather = zone.getActiveWeather();
FogOptions fogConfig = activeWeather.getFogOptions();

// When a player needs a state update, the config is converted to a packet
if (fogConfig != null) {
    com.hypixel.hytale.protocol.FogOptions packet = fogConfig.toPacket();
    player.getConnection().send(packet);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new FogOptions()`. This bypasses the asset loading system and creates a default, unconfigured object that does not reflect the server's intended state. Always rely on the Codec system to create instances from asset files.
- **Runtime Mutation:** Do not modify the fields of a FogOptions object after it has been loaded. This creates a discrepancy between the in-memory state and the source configuration files, leading to inconsistent behavior across server restarts.

## Data Pipeline
The FogOptions class is a key step in the pipeline that translates a static configuration file into a real-time rendering instruction for the client.

> Flow:
> Weather Asset File (JSON/HOCON) -> AssetManager -> **FogOptions.CODEC** -> **FogOptions Instance** -> WeatherService -> toPacket() -> Network Packet -> Client Rendering Engine

