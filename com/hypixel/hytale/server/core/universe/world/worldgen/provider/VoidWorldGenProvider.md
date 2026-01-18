---
description: Architectural reference for VoidWorldGenProvider
---

# VoidWorldGenProvider

**Package:** com.hypixel.hytale.server.core.universe.world.worldgen.provider
**Type:** Transient

## Definition
```java
// Signature
public class VoidWorldGenProvider implements IWorldGenProvider {
```

## Architecture & Concepts
The VoidWorldGenProvider is a specialized implementation of the IWorldGenProvider interface. Its sole purpose is to produce a world generator, IWorldGen, that creates completely empty, or "void", chunks. This component is fundamental for creating minimalist game worlds, such as server lobbies, skyblock-style maps, or controlled testing environments where standard terrain generation is unnecessary and computationally expensive.

Architecturally, this class follows a **Factory Pattern**. The VoidWorldGenProvider itself acts as a configurable factory object. It is designed to be deserialized from server configuration files via its static CODEC field. Once instantiated and configured with properties like a global tint and environment, its primary role is to be consumed once to create an instance of its inner class, VoidWorldGen. The VoidWorldGen object is the concrete generator that executes the logic to produce empty chunks on demand.

This separation of concerns is critical:
1.  **VoidWorldGenProvider:** The external, configurable definition of the void world. It bridges the gap between static server configuration and the live world generation system.
2.  **VoidWorldGen (inner class):** The internal, immutable, and thread-safe worker. It contains the pre-calculated IDs for tint and environment and executes the generation logic within the server's multithreaded world generation pipeline.

## Lifecycle & Ownership
-   **Creation:** An instance is created by the Hytale codec system during server startup when it parses the world's generation configuration. It is never instantiated directly in game logic.
-   **Scope:** This object is extremely short-lived. It exists only during the world initialization phase. Its lifetime is bound to the process of creating and registering the world's designated IWorldGen instance.
-   **Destruction:** It becomes eligible for garbage collection immediately after the `getGenerator` method is called and the resulting VoidWorldGen instance is passed to the world management system. It holds no references and is not retained.

## Internal State & Concurrency
-   **State:** The class holds two pieces of mutable state: `tint` (a Color object) and `environment` (a String). This state is populated by the codec system during deserialization. After this initial setup, the state is treated as immutable.
-   **Thread Safety:** This class is **not thread-safe**. It is designed for single-threaded access during the server's boot sequence. The returned VoidWorldGen instance, however, is **immutable and inherently thread-safe**. Its `generate` method can be safely invoked by multiple world generation worker threads concurrently without any risk of data races, as it contains no mutable state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getGenerator() | IWorldGen | O(1) | Factory method. Validates the configured state and constructs an immutable, thread-safe VoidWorldGen instance. Throws WorldGenLoadException if the specified environment name is not a valid, registered asset. |

## Integration Patterns

### Standard Usage
This provider is not used directly in code. Instead, it is declared within a world's JSON or YAML configuration file. The Hytale server parses this configuration at startup to build the appropriate world generator.

**Conceptual World Configuration Example:**
```yaml
# In a worldgen.yml or similar configuration file
generator:
  type: "Void"
  Tint: "#404060"
  Environment: "hytale:the_void"
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new VoidWorldGenProvider()`. The object is designed to be configured and created by the server's codec system. Manual creation bypasses this critical data-driven pipeline.
-   **State Mutation:** Do not modify the public fields of a VoidWorldGenProvider instance after it has been created by the codec. The `getGenerator` method is intended to be called on a finalized configuration.

## Data Pipeline
The primary data flow is from a static configuration file into a live, functional object within the engine.

> Flow:
> WorldGen Config File -> Server Codec System -> **VoidWorldGenProvider Instance** -> `getGenerator()` call -> `VoidWorldGen` Instance -> World Generation Engine

Once the `VoidWorldGen` instance is created, it participates in the chunk generation pipeline:

> Chunk Request -> `VoidWorldGen.generate()` -> New `GeneratedBlockChunk` -> Set Tint & Environment Data -> Return `CompletableFuture<GeneratedChunk>`

