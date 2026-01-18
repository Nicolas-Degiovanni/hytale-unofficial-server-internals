---
description: Architectural reference for WorldProvider
---

# WorldProvider

**Package:** com.hypixel.hytale.server.core.universe.world
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface WorldProvider {
```

## Architecture & Concepts
The WorldProvider interface defines a strict contract for any object that can supply a reference to a World instance. It serves as a critical abstraction layer within the server architecture, decoupling game logic systems from the concrete management and lifecycle of World objects.

This pattern allows various systems, such as entity management, physics simulation, or AI, to operate on a World without needing to know how that World was created, loaded, or whether it represents a persistent dimension, a temporary instance, or a test environment. Implementations of this interface are responsible for handling the specifics of world retrieval, which may involve complex operations like lazy loading from disk, generation, or lookup from an in-memory cache.

In essence, WorldProvider is the designated entry point for accessing the primary game state container for a specific server context.

### Lifecycle & Ownership
As an interface, WorldProvider itself has no lifecycle. The following pertains to the lifecycle of its concrete implementations.

- **Creation:** Implementations are typically instantiated and registered by a higher-level authority, such as a UniverseManager or a DimensionService, during server startup or when a new dimension is loaded. They are not intended for direct instantiation by game logic systems.
- **Scope:** A WorldProvider instance is scoped to the lifetime of the world it represents. It persists as long as its corresponding dimension is active and loaded in memory.
- **Destruction:** The provider object is eligible for garbage collection when the owning system (e.g., UniverseManager) unloads or discards the associated World. There is no explicit destruction method on the interface.

## Internal State & Concurrency
- **State:** This interface is inherently stateless. However, implementations are expected to be stateful. An implementation might cache the World reference, manage loading states, or contain configuration details for the world it provides.
- **Thread Safety:** The contract makes no guarantees about thread safety. Implementations are **not** assumed to be thread-safe. Accessing a WorldProvider from multiple threads must be externally synchronized. The returned World object is highly stateful and is definitively not thread-safe; all mutations to the World must occur on the main server thread.

## API Surface
The public contract is minimal, focusing exclusively on world retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWorld() | World | Variable | Retrieves the World instance. This call may block if the world is not yet loaded. Guaranteed to return a non-null object. |

## Integration Patterns

### Standard Usage
The provider should be retrieved from a governing context or dependency injection system. Game systems should then call getWorld to retrieve a reference to the world for the current tick or operation.

```java
// Correctly retrieve the provider for the current context and get the world
WorldProvider provider = context.getService(WorldProvider.class);
World currentWorld = provider.getWorld();
currentWorld.updateEntities();
```

### Anti-Patterns (Do NOT do this)
- **Caching the World Reference:** Do not call getWorld() once and store the result in a long-lived field. The provider's internal logic may change, or the underlying world instance could be swapped. Always request the world from the provider when you need to operate on it.
- **Assuming Implementation:** Never cast a WorldProvider to a concrete class to access implementation-specific methods. This violates the abstraction and creates brittle code that will break if the world management strategy changes.

## Data Pipeline
WorldProvider acts as a source, not a pipeline component. It is the starting point for any system that needs to read from or write to the primary game state.

> Flow:
> Game Logic System -> **WorldProvider.getWorld()** -> World Object -> State Query / Mutation

