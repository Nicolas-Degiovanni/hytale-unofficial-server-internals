---
description: Architectural reference for ExplosionUtils
---

# ExplosionUtils

**Package:** com.hypixel.hytale.server.core.entity
**Type:** Utility

## Definition
```java
// Signature
public class ExplosionUtils {
```

## Architecture & Concepts

The ExplosionUtils class is a stateless, server-side physics and game logic calculator. It provides a centralized, high-fidelity implementation for all explosive effects within the game world. It is not an Entity Component System (ECS) system itself, but rather a utility invoked by other systems to enact complex world and entity mutations.

The core architectural concept is **occlusion-based damage propagation**. Unlike a naive implementation that damages all targets within a spherical radius, ExplosionUtils simulates a blast wave by performing hundreds of line-of-sight checks. It effectively ray-casts from the explosion's epicenter to surrounding blocks and entities. A target is only affected if it has a clear line of sight, meaning blocks and other obstructions can absorb the blast, creating a more realistic and computationally intensive simulation.

All world and entity state modifications are funneled through a CommandBuffer. This design is critical for engine stability, ensuring that all changes are queued and applied at a safe synchronization point within the server's main tick loop, thus preventing race conditions and maintaining data integrity in a concurrent environment.

### Lifecycle & Ownership
- **Creation:** Not applicable. As a static utility class, ExplosionUtils is never instantiated. Its methods are loaded into the JVM by the class loader at runtime.
- **Scope:** Static. The class and its methods are available for the entire lifetime of the server process.
- **Destruction:** Not applicable. The class is unloaded only when the server shuts down and its class loader is garbage collected.

## Internal State & Concurrency
- **State:** Stateless. ExplosionUtils contains no static or instance fields that persist data between invocations. All required state is provided as method arguments, making its behavior entirely deterministic based on its inputs.

- **Thread Safety:** This class is inherently thread-safe. Its stateless design prevents data corruption from concurrent calls. To manage randomness without contention, it utilizes ThreadLocalRandom.

    **WARNING:** While the utility itself is thread-safe, its primary function is to queue operations on a CommandBuffer. The thread safety of the overall explosion process is therefore dependent on the correct implementation and handling of the CommandBuffer by the calling system. Direct modification of world state is not performed and would violate engine concurrency patterns.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| performExplosion(source, pos, config, ignore, cmdBuffer, chunks) | void | O(r³) | The primary entry point. Orchestrates the entire explosion simulation, including block destruction, entity damage, and knockback. Complexity is cubic in relation to the explosion radius *r*. |

## Integration Patterns

### Standard Usage

ExplosionUtils MUST be called from a server-side system that has access to the current tick's CommandBuffer. The caller is responsible for constructing the Damage.Source and ExplosionConfig objects that define the explosion's properties.

```java
// Example from within an ECS System update method
ExplosionConfig config = buildExplosionConfig();
Vector3d explosionCenter = entityTransform.getPosition();
Damage.Source damageSource = Damage.Source.fromEntity(sourceEntityRef);

// The core invocation
ExplosionUtils.performExplosion(
    damageSource,
    explosionCenter,
    config,
    sourceEntityRef, // Entity to ignore (e.g., the creeper itself)
    commandBuffer,   // The current tick's command buffer
    chunkStoreAccessor
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** This is a static utility class. Attempting to instantiate it with `new ExplosionUtils()` is impossible and indicates a misunderstanding of its design.
- **Large Radii:** Invoking performExplosion with an excessively large radius in the ExplosionConfig can cause severe server performance degradation (tick lag). The O(r³) complexity means that doubling the radius can increase the computational cost by a factor of eight.
- **Null CommandBuffer:** Passing a null CommandBuffer will result in a NullPointerException. This utility is fundamentally coupled to the command buffer pattern for all state mutations.

## Data Pipeline

The data flow for an explosion is a multi-stage process orchestrated entirely within the performExplosion method. The function acts as a pipeline that transforms a high-level event into a series of low-level, deferred state changes.

> Flow:
> Game Event (e.g., TNT ignition) -> **ExplosionUtils.performExplosion** -> Sphere Generation -> Per-Block Ray-casting -> Occlusion & Line-of-Sight Checks -> Damage & Knockback Calculation -> Queued operations in **CommandBuffer** -> Synchronized Game State Update in next tick phase

