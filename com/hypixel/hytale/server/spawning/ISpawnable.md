---
description: Architectural reference for ISpawnable
---

# ISpawnable

**Package:** com.hypixel.hytale.server.spawning
**Type:** Contract Interface

## Definition
```java
// Signature
public interface ISpawnable {
```

## Architecture & Concepts
The ISpawnable interface is a fundamental contract within the server-side entity spawning framework. It establishes a standardized protocol for any object definition that can be instantiated into the game world. Its primary architectural purpose is to decouple the core spawning engine from the specific rules and conditions of individual spawnable entities, such as creatures, NPCs, or even dynamic environmental objects.

By implementing this interface, an entity definition provides the spawning system with two critical pieces of information: a unique identifier for lookup and a predicate method, canSpawn, to determine if the current world state is suitable for its creation. This pattern allows for a highly extensible system where new spawnable types can be added without modifying the central spawning logic. The system queries a registry of ISpawnable implementations, passing a SpawningContext to each, to make dynamic decisions about world population.

## Lifecycle & Ownership
As an interface, ISpawnable itself does not have a lifecycle. Instead, it defines a contract that its implementing classes must adhere to. The lifecycle and ownership concerns apply to the concrete objects that implement this interface.

- **Creation:** Implementations of ISpawnable are typically created and registered during the server's bootstrap or content loading phase. They are often managed by a central registry or manager, such as an EntityTypeRegistry.
- **Scope:** These implementation objects are long-lived, typically persisting for the entire duration of the server session. They act as stateless prototypes or factories for the entities they represent.
- **Destruction:** They are destroyed when the server shuts down or when game content is unloaded, as part of the registry's cleanup process.

**WARNING:** The spawning system holds references to ISpawnable implementations but does not own them. Ownership is managed by the registry that provides them.

## Internal State & Concurrency
- **State:** The ISpawnable contract is designed to be implemented by stateless objects. The canSpawn method should not modify the internal state of the implementing object. All necessary world state for decision-making is provided via the SpawningContext argument.
- **Thread Safety:** Implementations of this interface are not guaranteed to be invoked from any specific thread, though they are most commonly called from the main server tick loop or a dedicated world generation thread. Implementations must be thread-safe. The canSpawn method should be a pure function of its inputs and avoid side effects. Any access to shared, mutable state must be properly synchronized.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getIdentifier() | String | O(1) | Returns a unique, non-null string identifier for the spawnable type. This is used for lookups, logging, and configuration. |
| canSpawn(SpawningContext) | SpawnTestResult | Variable | Evaluates the provided world context to determine if this object can be spawned. Complexity depends on the implementation's rules (e.g., biome checks, light level sampling, proximity checks). |

## Integration Patterns

### Standard Usage
The primary integration pattern is to implement this interface on a class that defines a type of entity. This class is then registered with a central spawning or entity type manager.

```java
// An entity type definition implementing the contract
public class KweebecType implements ISpawnable {
    @Override
    public String getIdentifier() {
        return "hytale:kweebec";
    }

    @Override
    public SpawnTestResult canSpawn(SpawningContext context) {
        // Logic to check for forest biomes, light levels, etc.
        if (context.isBiome(Biomes.FOREST) && context.getLightLevel() > 7) {
            return SpawnTestResult.SUCCESS;
        }
        return SpawnTestResult.FAILURE_BIOME;
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Blocking Operations:** Do not perform file I/O, database queries, or other long-running, blocking operations within the canSpawn method. This method may be called thousands of times per tick and will directly impact server performance.
- **Stateful Implementations:** Avoid storing mutable state within the ISpawnable implementation itself. This can lead to unpredictable behavior and race conditions if the same instance is used across multiple spawning evaluations.
- **Non-Unique Identifiers:** The value returned by getIdentifier must be unique across all registered ISpawnable types. Collisions will lead to undefined behavior in the spawning system.

## Data Pipeline
The ISpawnable interface acts as a decision point or a filter within the larger entity spawning data pipeline.

> Flow:
> Spawning System Tick -> Identify Spawn Location -> Create SpawningContext -> Query Registry for **ISpawnable** -> **ISpawnable.canSpawn(Context)** -> Receive SpawnTestResult -> If SUCCESS, request Entity Factory -> Instantiate Entity -> Insert Entity into World

