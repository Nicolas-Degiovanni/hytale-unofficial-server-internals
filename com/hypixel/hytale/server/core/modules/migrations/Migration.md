---
description: Architectural reference for the Migration interface, defining a contract for world data upgrades.
---

# Migration

**Package:** com.hypixel.hytale.server.core.modules.migrations
**Type:** Contract / Strategy Pattern

## Definition
```java
// Signature
public interface Migration {
   void run(WorldChunk var1);
}
```

## Architecture & Concepts
The Migration interface defines a contract for a single, atomic data transformation applied to a WorldChunk. It is a core component of the server's data versioning and upgrade system, designed to ensure forward compatibility of world data as the game evolves.

This interface embodies the **Strategy Pattern**, decoupling the specific logic of a data upgrade (the *what*) from the orchestration of applying those upgrades (the *how*). A central migration service is responsible for discovering all implementations of Migration, determining which ones need to be applied to a given chunk based on its stored data version, and executing them in a controlled, sequential order.

This architecture is critical for preventing world corruption and allows developers to introduce breaking changes to data formats without requiring a full world reset. Each implementation represents a single, idempotent step in a larger upgrade chain.

### Lifecycle & Ownership
- **Creation:** Implementations of Migration are expected to be discovered and instantiated by a central MigrationService during server bootstrap, likely via classpath scanning or a service registry. They are not intended for manual instantiation.
- **Scope:** Instances are typically stateless singletons, persisting for the entire lifetime of the server process. Their purpose is to hold logic, not data.
- **Destruction:** De-referenced and garbage collected during server shutdown when the managing service is destroyed.

## Internal State & Concurrency
- **State:** As an interface, Migration is stateless. Implementations **must** be designed to be stateless. All necessary data for the transformation must be contained within the provided WorldChunk argument. Storing instance-level state across multiple `run` calls is a severe anti-pattern that will lead to unpredictable behavior in a multi-threaded world loading environment.
- **Thread Safety:** The `run` method is guaranteed to have exclusive access to a specific WorldChunk instance during its execution. However, the server may process multiple chunks on different threads concurrently. Therefore, implementations **must be re-entrant** and must not rely on or modify any shared global state without proper synchronization. The safest design is a pure function that only modifies the input argument.

## API Surface
The public contract consists of a single method, defining the core operation of the pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| run(WorldChunk chunk) | void | O(N) | Executes the data transformation on the provided chunk in-place. Complexity is dependent on the implementation but is typically proportional to the amount of data in the chunk. |

## Integration Patterns

### Standard Usage
Developers should not invoke Migration implementations directly. Instead, they create a new class implementing the interface and rely on the server's migration service to execute it at the appropriate time.

```java
// Example of a new migration implementation.
// The system will automatically discover and run this.
public class V2_UpdateBlockIdMigration implements Migration {
    private static final int OLD_BLOCK_ID = 123;
    private static final int NEW_BLOCK_ID = 456;

    @Override
    public void run(WorldChunk chunk) {
        // Logic to iterate through the chunk and replace
        // an old block ID with a new one.
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Invocation:** Never call the `run` method directly. This bypasses the server's version tracking system and can lead to applying a migration out of order, multiple times, or not at all, resulting in data corruption.
- **Stateful Implementations:** Do not store chunk-specific or request-specific data as fields in your Migration class. This is not thread-safe and will fail when the server processes multiple chunks in parallel.
- **Complex Operations:** Avoid performing heavy computations, network requests, or blocking file I/O within a migration. Migrations are on the critical path for chunk loading and must be highly performant. They are intended for data transformation only.

## Data Pipeline
The Migration interface sits squarely in the middle of the world chunk loading process, acting as a conditional transformation stage.

> Flow:
> Disk Read -> Chunk Deserialization -> Server Version Check -> **Migration.run()** (If Needed) -> Chunk Provided to Game Engine

