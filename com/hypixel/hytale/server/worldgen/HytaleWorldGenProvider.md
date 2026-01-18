---
description: Architectural reference for HytaleWorldGenProvider
---

# HytaleWorldGenProvider

**Package:** com.hypixel.hytale.server.worldgen
**Type:** Transient Configuration

## Definition
```java
// Signature
public class HytaleWorldGenProvider implements IWorldGenProvider {
```

## Architecture & Concepts
The HytaleWorldGenProvider serves as a configuration-driven factory for creating concrete world generator instances. It is not the world generator itself; rather, it is a descriptor that resolves and loads the appropriate generator based on parameters defined in server configuration files.

Its primary role is to bridge the gap between a high-level, human-readable configuration (e.g., a JSON file specifying a world generator by name) and the low-level `IWorldGen` implementation required by the server's `Universe`.

The class is designed for deserialization. The static `CODEC` field exposes a schema that allows the server's configuration loader to instantiate and populate a HytaleWorldGenProvider object from a data source. The core logic within `getGenerator` involves resolving file paths, handling a "Default" case, and delegating the actual loading process to a `ChunkGeneratorJsonLoader`. This isolates path resolution and configuration logic from the core world generation machinery.

### Lifecycle & Ownership
- **Creation:** Instances are not created directly via the constructor. They are instantiated by the `BuilderCodec` system when the server deserializes a world's configuration. This process is typically triggered by the `Universe` during world initialization.
- **Scope:** The object has a very short, transient lifecycle. It exists only for the duration of the `getGenerator` method call. Once the `IWorldGen` instance is retrieved, the HytaleWorldGenProvider that created it is no longer referenced and becomes eligible for garbage collection.
- **Destruction:** Managed entirely by the Java garbage collector. There are no native resources or explicit cleanup methods.

## Internal State & Concurrency
- **State:** The internal state consists of the `name` and `path` strings, which are mutable only during the deserialization process managed by the `CODEC`. After instantiation, the object should be treated as immutable. It holds configuration data and does not cache any results.
- **Thread Safety:** This class is **not thread-safe**. However, its intended use pattern—instantiate, use once, and discard—obviates the need for synchronization. It is designed to be used by a single thread during the world loading sequence. Calling `getGenerator` from multiple threads on a shared instance would be a severe design error.

## API Surface
The public contract is minimal, centered on the `IWorldGenProvider` interface and the static `CODEC` for deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getGenerator() | IWorldGen | I/O Bound | Resolves the configured path and name, then instantiates and loads a world generator from disk. Throws `WorldGenLoadException` on failure. |
| CODEC | BuilderCodec | N/A | A static schema definition used by the server to deserialize this object from a configuration file. |

## Integration Patterns

### Standard Usage
The HytaleWorldGenProvider is used declaratively in a configuration file. The server's loading mechanism uses the associated `CODEC` to create an instance and retrieve the final generator. Direct interaction is rare.

```java
// Pseudo-code demonstrating the server's internal flow

// 1. A configuration string is read from a world settings file
String worldGenConfig = "{ \"Type\": \"Hytale\", \"Name\": \"MyCustomWorld\", \"Path\": \"/path/to/custom/worldgen\" }";

// 2. The server uses the codec registry to find the right codec and deserialize
IWorldGenProvider provider = Codec.deserialize(worldGenConfig); // This creates a HytaleWorldGenProvider

// 3. The provider is used to get the actual generator
IWorldGen worldGenerator = provider.getGenerator();

// 4. The provider instance is now out of scope and can be GC'd
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new HytaleWorldGenProvider()`. The resulting object will lack the necessary `name` and `path` configuration and will likely fail or load the incorrect generator. Always rely on the `CODEC`-based deserialization process.
- **State Mutation:** Do not modify the public fields of this class after it has been created. It is intended to be a read-only configuration object post-deserialization.
- **Caching Instances:** Do not hold a long-term reference to a HytaleWorldGenProvider instance. Its purpose is fulfilled the moment `getGenerator` returns.

## Data Pipeline
This component acts as an early stage in the world generation loading pipeline, responsible for interpreting configuration and initiating the load process.

> Flow:
> Server Configuration File -> Codec Deserialization -> **HytaleWorldGenProvider Instance** -> `getGenerator()` call -> ChunkGeneratorJsonLoader -> `IWorldGen` Instance -> Universe World Ticker

