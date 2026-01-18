---
description: Architectural reference for DummyWorldGenProvider
---

# DummyWorldGenProvider

**Package:** com.hypixel.hytale.server.core.universe.world.worldgen.provider
**Type:** Transient

## Definition
```java
// Signature
public class DummyWorldGenProvider implements IWorldGenProvider {
```

## Architecture & Concepts
The DummyWorldGenProvider is a concrete implementation of the IWorldGenProvider interface. It serves as a foundational, minimal-effort world generator, primarily used for testing, debugging, or as a fallback mechanism when a more complex generator is not configured.

Architecturally, it represents a pluggable strategy within the server's world generation subsystem. The server does not hard-code world generation logic; instead, it relies on providers that can be selected and configured at runtime, typically via world configuration files. This class, identified by the static ID "Dummy", is the simplest possible strategy: it generates a single, flat layer of blocks.

Its primary role is to provide an instance of IWorldGen (the nested DummyWorldGen class) which performs the actual chunk generation. This two-step pattern (Provider -> Generator) allows the provider to be a lightweight, configurable factory, while the generator instance can potentially hold state for a specific world seed, though this dummy implementation remains stateless.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly. The class is designed to be instantiated by the server's codec and registry system, which deserializes world configuration data. The static CODEC field is the designated entry point for this data-driven creation process.
- **Scope:** An instance of DummyWorldGenProvider is typically short-lived. The world generation service may create it, immediately call getGenerator to retrieve the IWorldGen instance, and then discard the provider. The returned DummyWorldGen instance is also transient and created anew on each call to getGenerator.
- **Destruction:** The object is stateless and requires no explicit cleanup. It is garbage collected when all references are lost.

## Internal State & Concurrency
- **State:** This class and its nested DummyWorldGen are entirely stateless and immutable. They contain no instance fields and their behavior is determined solely by the parameters passed to their methods.
- **Thread Safety:** The class is inherently thread-safe. As it holds no state, instances can be shared and methods can be invoked from any thread without synchronization. The generate method returns a CompletableFuture, signaling its design for use within an asynchronous, multi-threaded chunk generation pipeline.

## API Surface
The public contract is fulfilled by implementing the IWorldGenProvider interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getGenerator() | IWorldGen | O(1) | Factory method. Returns a new, stateless DummyWorldGen instance. |

The returned IWorldGen object has the following critical methods:

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSpawnPoints(seed) | Transform[] | O(1) | Returns a hard-coded, default spawn point. Ignores the seed. |
| generate(...) | CompletableFuture | O(1) | Generates a single chunk with a flat plane of blocks at Y=0. The operation is constant time as chunk dimensions are fixed. Returns an already-completed future. |

## Integration Patterns

### Standard Usage
This provider is not intended for direct programmatic use. It is designed to be specified within a world configuration file, which is then loaded by the server at startup.

**Example World Configuration (Conceptual JSON):**
```json
{
  "worldGen": {
    "provider": "Dummy"
  }
}
```
The server's world management system would parse this, find the provider registered with the ID "Dummy", and use its codec to instantiate it.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new DummyWorldGenProvider()`. This bypasses the server's configuration and registration system, making the system brittle and difficult to manage.
- **Production Use:** This provider generates a non-playable, empty world. Using it for a live game server is an incorrect application of its purpose. It is strictly for development and testing.
- **Extension:** Do not extend this class to add complex world generation features. Instead, create a new, separate implementation of the IWorldGenProvider interface.

## Data Pipeline
The DummyWorldGenProvider acts as a simple, synchronous source in the asynchronous world generation pipeline.

> Flow:
> Chunk Generation Request -> World Generation Service -> **DummyWorldGenProvider.getGenerator()** -> IWorldGen.generate() -> CompletedFuture(GeneratedChunk) -> Chunk Population System

