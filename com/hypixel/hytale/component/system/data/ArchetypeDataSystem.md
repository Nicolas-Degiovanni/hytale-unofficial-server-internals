---
description: Architectural reference for ArchetypeDataSystem
---

# ArchetypeDataSystem

**Package:** com.hypixel.hytale.component.system.data
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class ArchetypeDataSystem<ECS_TYPE, Q, R> extends System<ECS_TYPE> implements QuerySystem<ECS_TYPE> {
```

## Architecture & Concepts

The ArchetypeDataSystem is a foundational abstract class within Hytale's Entity Component System (ECS) framework. It establishes a standardized, high-performance contract for systems whose primary responsibility is to **query and retrieve data** from the component world. It is not a concrete system but rather a template that enforces a critical architectural pattern.

This class acts as the primary read-only data access layer for game logic. Its design is heavily optimized for performance by operating on an ArchetypeChunk, a contiguous block of memory containing components for entities of the same type. This data-oriented approach ensures cache-friendly memory access patterns, which is critical for performance in a large-scale simulation.

The key architectural contributions are:
- **Separation of Concerns:** It cleanly separates the logic of *how to find data* from the logic of *what to do with data*.
- **Performance:** By enforcing iteration over ArchetypeChunks, it guides developers towards writing highly efficient, cache-coherent code.
- **Flexibility:** The generic parameters Q (Query) and R (Result) allow for the creation of highly specific, type-safe query systems for any data retrieval need.
- **Concurrency Model:** The inclusion of a CommandBuffer in its primary method signature is a deliberate design choice to support parallel execution. Systems can query the world state and enqueue state-modifying commands without causing data races.

## Lifecycle & Ownership

- **Creation:** Concrete implementations of ArchetypeDataSystem are not instantiated directly in game logic. They are discovered and instantiated by the central SystemRegistry during the initialization of an ECS world.
- **Scope:** An instance of a data system persists for the entire lifetime of the ECS world it belongs to. Its lifecycle is directly coupled to the game session or server instance.
- **Destruction:** The system is marked for garbage collection when its parent ECS world is torn down, for example, when a player leaves a server or the server shuts down.

## Internal State & Concurrency

- **State:** This abstract class is stateless. Concrete implementations are **strongly expected to be stateless**. They should not contain mutable instance fields that persist across invocations of the fetch method. All necessary context is provided through the method arguments.
- **Thread Safety:** The fetch method is designed to be executed concurrently by the engine's task scheduler. Implementations must adhere to a strict concurrency model:
    1. Reading from the Store and ArchetypeChunk is safe.
    2. Writing to the provided result List is safe, as this list is typically thread-local or pre-allocated for the calling thread.
    3. **All world mutations must be deferred** by writing to the CommandBuffer. Direct modification of the Store or its components from within fetch will lead to severe data corruption and is strictly forbidden.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fetch(chunk, store, commands, query, results) | void | O(N) | Executes the data retrieval logic. Iterates entities in the provided ArchetypeChunk, applies query criteria, and populates the results list. N is the number of entities in the chunk. |

## Integration Patterns

### Standard Usage

Developers do not call this system directly. Instead, they create a concrete implementation that defines a specific query. The engine's SystemExecutor is responsible for invoking the fetch method on these systems.

```java
// A concrete implementation to find all entities with a Health component
// that have less than a specified amount of health.

// 1. Define the Query and Result data structures
public class HealthQuery {
    public final int maxHealth;
    public HealthQuery(int maxHealth) { this.maxHealth = maxHealth; }
}

public class LowHealthResult {
    public final EcsId entityId;
    public final int currentHealth;
    // ... constructor
}

// 2. Implement the ArchetypeDataSystem
public class FindLowHealthSystem extends ArchetypeDataSystem<EcsId, HealthQuery, LowHealthResult> {
    @Override
    public void fetch(ArchetypeChunk<EcsId> chunk, Store<EcsId> store, CommandBuffer<EcsId> commands, HealthQuery query, List<LowHealthResult> results) {
        // Assume the chunk is known to contain Health components
        ComponentArray<Health> healthComponents = chunk.getComponentArray(Health.class);

        for (int i = 0; i < chunk.size(); i++) {
            Health health = healthComponents.get(i);
            if (health.value < query.maxHealth) {
                EcsId entityId = chunk.getEntityId(i);
                results.add(new LowHealthResult(entityId, health.value));
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)

- **Direct Mutation:** Never modify components retrieved from the Store or ArchetypeChunk within the fetch method. This bypasses the CommandBuffer and will break thread safety and data consistency.
    ```java
    // DANGEROUS: Direct mutation inside a fetch call
    Health health = healthComponents.get(i);
    if (health.value <= 0) {
        health.value = 0; // ANTI-PATTERN! Causes race conditions.
    }
    ```
- **Stateful Implementations:** Storing state in instance variables across calls is not thread-safe and produces unpredictable behavior when the system is executed in parallel.
    ```java
    // DANGEROUS: Stateful system
    public class BadSystem extends ArchetypeDataSystem<...> {
        private int entitiesFound = 0; // ANTI-PATTERN!

        @Override
        public void fetch(...) {
            // ... logic
            this.entitiesFound++; // Unsafe in a multithreaded context
        }
    }
    ```

## Data Pipeline

The ArchetypeDataSystem is a processing stage within a larger query execution pipeline orchestrated by the engine.

> Flow:
> High-Level Logic (e.g., AI Behavior Tree) -> Submits Query (Q) -> System Executor -> **ArchetypeDataSystem.fetch** -> Iterates ArchetypeChunk -> Populates Result List (R) -> Results returned to High-Level Logic

