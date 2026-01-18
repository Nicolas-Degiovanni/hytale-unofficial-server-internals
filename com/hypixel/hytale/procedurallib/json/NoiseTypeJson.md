---
description: Architectural reference for NoiseTypeJson
---

# NoiseTypeJson

**Package:** com.hypixel.hytale.procedurallib.json
**Type:** Singleton Enum / Factory

## Definition
```java
// Signature
public enum NoiseTypeJson {
    // ... enum constants
}
```

## Architecture & Concepts
The **NoiseTypeJson** enum serves as a critical factory and dispatcher within the procedural generation framework. Its primary responsibility is to decouple the high-level configuration parsing from the concrete implementation of noise function loaders. It implements a variation of the Strategy and Factory patterns using Java's enum construct.

Each enum constant (e.g., PERLIN, SIMPLEX, CELL) represents a specific type of noise function and holds a direct, reflective reference to the corresponding **JsonLoader** class responsible for parsing its configuration. When the system needs to instantiate a noise function from a JSON definition, it uses this enum to look up the correct loader type and create a new instance of it.

This design centralizes the mapping of noise types to their loaders, making the system extensible. To support a new noise type, a developer only needs to add a new entry to this enum and provide the associated loader class, without modifying the core parsing logic. The use of reflection via `Constructor.newInstance` is a key implementation detail that enables this dynamic instantiation.

## Lifecycle & Ownership
- **Creation:** All instances of the **NoiseTypeJson** enum (CELL, CONSTANT, etc.) are instantiated by the Java Virtual Machine's class loader when the enum is first accessed. This is a one-time, static initialization that occurs early in the application lifecycle. The constructor logic, which uses reflection to cache the loader's constructor, is executed only during this initial phase.
- **Scope:** As static final instances, the enum constants persist for the entire lifetime of the application. They are globally accessible and their state is immutable post-initialization.
- **Destruction:** The enum instances are garbage collected only when the JVM shuts down. No manual cleanup is required or possible.

## Internal State & Concurrency
- **State:** The internal state of each enum constant is immutable. It consists of a **NoiseType** and a cached **Constructor** object. This state is established during static initialization and never changes.
- **Thread Safety:** This class is inherently thread-safe. Its immutable state guarantees that concurrent reads from multiple threads are safe. The **newLoader** method is also thread-safe as it creates a new, distinct **JsonLoader** object on every invocation, preventing any state interference between calls.

**WARNING:** While **NoiseTypeJson** itself is thread-safe, the **JsonLoader** instances it creates may not be. Concurrency guarantees of the returned objects are the responsibility of the specific loader implementation.

## API Surface
The public contract is minimal, exposing a single factory method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| newLoader(seed, dataFolder, json) | JsonLoader | O(1) | Instantiates and returns a new loader configured for the specific noise type. Throws a critical **Error** if the loader's constructor cannot be found or fails to execute, indicating a severe configuration or build issue. |

## Integration Patterns

### Standard Usage
The intended use is for a higher-level configuration service to parse a JSON object, identify the noise type string, resolve it to the corresponding **NoiseTypeJson** constant, and then invoke the factory method to get the correct loader.

```java
// Assume 'configElement' is a JsonElement from a config file
// and 'noiseTypeString' is a field like "PERLIN"
JsonElement configElement = ...;
String noiseTypeString = configElement.getAsJsonObject().get("type").getAsString();

// Resolve the string to the enum constant
NoiseTypeJson noiseType = NoiseTypeJson.valueOf(noiseTypeString);

// Use the enum as a factory to get the correct loader
JsonLoader<?, NoiseFunction> loader = noiseType.newLoader(seed, dataPath, configElement);

// The loader can now be used to produce the final NoiseFunction
NoiseFunction finalNoise = loader.load();
```

### Anti-Patterns (Do NOT do this)
- **Catching Errors:** The **newLoader** method throws an **Error**, not an **Exception**. This signals a fatal, unrecoverable state, such as a missing loader class or a constructor with an incorrect signature. Attempting to catch and suppress this **Error** will mask fundamental programming or deployment mistakes and lead to system instability.
- **Modifying via Reflection:** Do not attempt to use reflection to modify the internal `constructor` or `noiseType` fields of the enum constants. This would violate the immutability contract and result in undefined behavior across the application.

## Data Pipeline
**NoiseTypeJson** acts as a dynamic factory in the data pipeline that transforms declarative JSON configuration into executable **NoiseFunction** objects.

> Flow:
> JSON File -> JSON Parser -> **NoiseTypeJson** Factory -> Specific **JsonLoader** -> **NoiseFunction** Object -> Procedural Engine

---

