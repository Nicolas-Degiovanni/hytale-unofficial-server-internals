---
description: Architectural reference for DistortedShapes
---

# DistortedShapes

**Package:** com.hypixel.hytale.server.worldgen.cave.shape.distorted
**Type:** Utility / Registry

## Definition
```java
// Signature
public final class DistortedShapes {
```

## Architecture & Concepts
The DistortedShapes class serves as a central, static registry for mapping string identifiers to `DistortedShape.Factory` implementations. Its primary architectural role is to decouple the cave generation system from the concrete implementations of cave shapes. This allows the world generator to request shape-creation logic by name (e.g., "PIPE", "CYLINDER") without being tightly coupled to specific factory classes like DistortedPipeShape.Factory.

This implementation of the Registry pattern is fundamental to the extensibility of the world generator. It provides a single, authoritative source for available cave shapes and allows for new shapes to be added at runtime, for instance during server initialization or mod loading.

The choice of a `ConcurrentHashMap` for the internal registry is a critical design decision, indicating that the system is built to support parallel initialization. Shape registration can occur from multiple threads without explicit locking, ensuring robust and efficient server startup.

### Lifecycle & Ownership
- **Creation:** This class is a static utility and is never instantiated; its constructor is private. Its static state, including the internal shape map, is initialized by the JVM ClassLoader when the class is first referenced. The static initializer block immediately populates the registry with the engine's default shapes (PIPE, CYLINDER, ELLIPSOID).
- **Scope:** The registry is static and therefore global. It persists for the entire lifetime of the server's Java Virtual Machine process.
- **Destruction:** The registry and its contents are only cleared when the JVM shuts down. There is no public API for unregistering shapes or clearing the map.

## Internal State & Concurrency
- **State:** The primary state is the static `SHAPES` map, which is mutable and stores the mapping of names to shape factories. The class also holds public static final references to the default factories (CYLINDER, ELLIPSE, PIPE), which are themselves immutable.
- **Thread Safety:** This class is explicitly designed to be thread-safe. The internal use of `ConcurrentHashMap` guarantees that all read and write operations on the registry are safe to perform from multiple threads simultaneously. The `register` method leverages the atomic `putIfAbsent` operation to prevent race conditions during registration, ensuring that the first registration for a given name always wins.

## API Surface
The public API consists entirely of static methods for registering and retrieving shape factories.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| register(name, factory) | void | O(1) amortized | Atomically registers a factory. If a factory with the same name already exists, this operation has no effect. |
| getByName(name) | DistortedShape.Factory | O(1) | Retrieves a factory by its registered name. Returns null if no matching factory is found. |
| getOrDefault(name) | DistortedShape.Factory | O(1) | Retrieves a factory by name. If not found, returns the default factory (PIPE). This is the safest method for world generation logic. |
| getDefault() | DistortedShape.Factory | O(1) | Returns the hardcoded default PIPE factory. |
| forEach(consumer) | void | O(N) | Executes a given action for each registered shape. |

## Integration Patterns

### Standard Usage
The primary pattern is to register custom shapes during an initialization phase and retrieve them by name during world generation.

```java
// In a mod or engine initialization routine
// Creates a new, custom shape factory
DistortedShape.Factory customSphereFactory = new CustomSphereShape.Factory();

// Registers the factory under a unique name
DistortedShapes.register("CUSTOM_SPHERE", customSphereFactory);

// Elsewhere, in the world generation code...
// The generator can now request this shape by name
String shapeNameFromConfig = "CUSTOM_SPHERE";
DistortedShape.Factory factory = DistortedShapes.getOrDefault(shapeNameFromConfig);
DistortedShape shape = factory.create(config);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The class has a private constructor. Attempting to create an instance with `new DistortedShapes()` will result in a compile-time error and is conceptually incorrect.
- **Late Registration:** Do not register shapes after world generation has begun. While technically thread-safe, it can lead to non-deterministic behavior where a world chunk may or may not have access to the new shape depending on timing. All registrations should occur during a well-defined server startup phase.
- **Overwriting Defaults:** The `register` method uses `putIfAbsent`, so attempting to overwrite a default shape like "PIPE" will silently fail. This is by design to protect core engine functionality.

## Data Pipeline
DistortedShapes does not process a continuous stream of data. Instead, it acts as a configuration and dependency provider during distinct phases of the server lifecycle.

> Flow:
> Server Bootstrap / Mod Loading -> **DistortedShapes.register()** -> World Generation System -> **DistortedShapes.getOrDefault(name)** -> Shape Factory -> Cave Generation Logic

