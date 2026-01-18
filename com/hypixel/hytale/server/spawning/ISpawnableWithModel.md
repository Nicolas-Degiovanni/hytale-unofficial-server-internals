---
description: Architectural reference for ISpawnableWithModel
---

# ISpawnableWithModel

**Package:** com.hypixel.hytale.server.spawning
**Type:** Contract Interface

## Definition
```java
// Signature
public interface ISpawnableWithModel extends ISpawnable {
```

## Architecture & Concepts
The ISpawnableWithModel interface defines a contract for any server-side entity that can be spawned and possesses a dynamically determined visual model. It extends the base ISpawnable contract, adding a layer of specificity for entities whose appearance is not static. This is a cornerstone of Hytale's dynamic world, allowing entities to have different models based on context, such as biome, world state, or specific gameplay events.

This interface acts as a formal agreement between the core **Spawning System** and the entity's specific logic. When the Spawning System processes a request to create an entity that implements this contract, it uses the interface methods to query the entity for its required model name and execution context *before* the entity is fully instantiated in the world. This decouples the generic spawning logic from the specific, data-driven logic of individual NPCs or creatures.

The parameters ExecutionContext and Scope are critical. They provide the implementing object with a snapshot of the current game state, enabling highly contextual decision-making.

## Lifecycle & Ownership
As an interface, ISpawnableWithModel has no lifecycle of its own. Instead, it dictates the lifecycle requirements for any class that implements it.

- **Creation:** Concrete implementations are typically instantiated by configuration loaders or asset managers when parsing game data files (e.g., JSON or HOCON definitions for NPCs). They represent the *template* for a spawnable entity, not the in-world instance itself.
- **Scope:** These template objects are long-lived, often loaded at server startup and persisting until the server shuts down. They are effectively immutable definitions.
- **Destruction:** The template objects are garbage collected during server shutdown or when a zone or world is completely unloaded.

**WARNING:** The objects implementing this interface are definitions, not live entities. Do not store per-instance state (like current health) in them.

## Internal State & Concurrency
The interface itself is stateless. Any class that implements ISpawnableWithModel is expected to be thread-safe. Spawning can occur across multiple threads as different parts of the world are simulated.

- **State:** Implementations should be treated as immutable or effectively immutable. Their purpose is to return data based on the provided ExecutionContext and Scope, not to maintain their own mutable state.
- **Thread Safety:** All methods must be safe to call from concurrent threads. The Spawning System provides no guarantees of serialization. Any internal caching within an implementation must be managed with thread-safe constructs.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getSpawnModelName(ctx, scope) | String | O(N) | **CRITICAL:** Resolves the asset name for the entity's model. Complexity depends on the implementation's expression evaluation. May be null. |
| createModifierScope(ctx) | Scope | O(1) | Throws IllegalStateException by default. Intended for more complex spawnables; its presence here suggests a design constraint. |
| createExecutionScope() | Scope | O(1) | Creates a new, empty scope for the entity's execution context. Essential for isolating the entity's state. |
| markNeedsReload() | void | O(1) | A control signal used to invalidate any internal caches. Likely called by a hot-reloading system when game data changes. |
| isMemory(ctx, scope) | boolean | O(N) | Determines if this spawnable should be treated as a "memory" for AI purposes. |
| getMemoriesCategory(ctx, scope) | String | O(N) | Returns the category key for the AI memory system. |
| getMemoriesNameOverride(ctx, scope) | String | O(N) | Returns a specific name for the AI memory, overriding defaults. |
| getNameTranslationKey(ctx, scope) | String | O(N) | **CRITICAL:** Returns the localization key for the entity's display name. Must not be null. |

## Integration Patterns

### Standard Usage
This interface is not used directly by gameplay programmers. Instead, it is implemented by data-driven entity classes. The server's Spawning System consumes this interface when it needs to bring an entity into the world.

```java
// Hypothetical Spawning System Usage
void spawnEntity(ISpawnableWithModel definition, SpawnLocation loc) {
    ExecutionContext context = createSpawnContext(loc);
    Scope scope = definition.createExecutionScope();

    String modelName = definition.getSpawnModelName(context, scope);
    if (modelName == null) {
        // Handle error: entity has no model in this context
        return;
    }

    // ... proceed to load model and create entity instance
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not store session-specific or instance-specific data in a class that implements this interface. It is a shared template.
- **Blocking Operations:** Implementations of getSpawnModelName or other methods must not perform blocking I/O or heavy computation. These methods are called frequently and must be highly performant.
- **Ignoring Context:** Failing to use the provided ExecutionContext and Scope to make decisions defeats the purpose of the interface and will lead to static, non-reactive entity behavior.

## Data Pipeline
This interface is a key component in the entity creation data pipeline. It acts as a data source for the Spawning System.

> Flow:
> Spawner Trigger -> **ISpawnableWithModel.getSpawnModelName()** -> Asset Path -> Asset Manager -> Renderable Model -> World Entity Instantiation

