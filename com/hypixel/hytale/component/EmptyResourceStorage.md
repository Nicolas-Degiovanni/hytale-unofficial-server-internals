---
description: Architectural reference for EmptyResourceStorage
---

# EmptyResourceStorage

**Package:** com.hypixel.hytale.component
**Type:** Singleton

## Definition
```java
// Signature
public class EmptyResourceStorage implements IResourceStorage {
```

## Architecture & Concepts
The EmptyResourceStorage class is a specific implementation of the IResourceStorage interface that embodies the **Null Object Pattern**. Its fundamental purpose is to provide a non-operational, or "no-op", storage backend. This component acts as a default or fallback strategy when no actual data persistence is required for a given resource.

In the engine's architecture, this class ensures that systems interacting with the IResourceStorage contract do not need to handle null references. Instead of configuring a null storage layer, the system can be configured with EmptyResourceStorage. This simplifies client code and prevents potential null pointer exceptions when dealing with entities or components that are transient and should not be saved to or loaded from a persistent medium like a disk or database.

It is commonly used for:
- Client-side cosmetic entities that have no server-side state.
- Temporary game objects created for animations or effects.
- Scenarios in a development or testing environment where persistence is disabled.

## Lifecycle & Ownership
- **Creation:** As a singleton, the single INSTANCE is created eagerly by the Java Class Loader when the EmptyResourceStorage class is first accessed. It is not instantiated by any dependency injection framework or factory.
- **Scope:** The object's lifecycle is tied to the application's lifecycle. It persists from the moment it is loaded by the class loader until the JVM is shut down.
- **Destruction:** The instance is garbage collected along with all other static objects during JVM shutdown. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** EmptyResourceStorage is **stateless and immutable**. It contains no instance fields and its behavior is constant. All operations are self-contained and do not modify any internal state.
- **Thread Safety:** This class is inherently **thread-safe**. Due to its stateless nature, all of its methods can be invoked concurrently from any number of threads without requiring locks or any other synchronization mechanisms.

## API Surface
The API surface exclusively consists of no-op implementations of the IResourceStorage contract. All methods return an already completed CompletableFuture, ensuring that any asynchronous call chain proceeds immediately without blocking or performing I/O.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | EmptyResourceStorage | O(1) | Returns the static, shared instance of the class. |
| load(store, data, resourceType) | CompletableFuture<T> | O(1) | Immediately returns a future containing a new, default-state resource. Does not read from any source. |
| save(store, data, resourceType, resource) | CompletableFuture<Void> | O(1) | No-operation. Immediately returns a completed future, effectively discarding the provided resource. |
| remove(store, data, resourceType) | CompletableFuture<Void> | O(1) | No-operation. Immediately returns a completed future. |

## Integration Patterns

### Standard Usage
EmptyResourceStorage is not intended for direct use in typical game logic. Instead, it is configured at a system level as the storage strategy for components or entities that should not be persisted. The engine's resource management systems will then transparently use this implementation.

```java
// Example: Configuring a component type to not be persisted
// This is conceptual and likely happens deep within engine setup code.
ComponentType nonPersistentComponent = new ComponentTypeBuilder()
    .withStorageStrategy(EmptyResourceStorage.get())
    .build();

// Later, when the engine tries to save an entity with this component,
// the call to save() on EmptyResourceStorage does nothing.
```

### Anti-Patterns (Do NOT do this)
- **Expecting Persistence:** Do not assign this storage strategy to components that hold critical state, such as player inventory or world data. Using EmptyResourceStorage for such components will lead to silent data loss, as save operations are ignored and load operations return new, empty objects.
- **Direct Instantiation:** Do not call `new EmptyResourceStorage()`. This violates the singleton pattern and creates unnecessary objects. Always use the static `EmptyResourceStorage.get()` method to retrieve the shared instance.

## Data Pipeline
The data pipeline for this component is terminal. It acts as a sink for save/remove operations and a factory for load operations.

> **Save/Remove Flow:**
> Resource Manager -> **EmptyResourceStorage.save()** -> Data is immediately discarded.

> **Load Flow:**
> Resource Manager -> **EmptyResourceStorage.load()** -> A new, default-constructed resource is created -> Resource Manager

