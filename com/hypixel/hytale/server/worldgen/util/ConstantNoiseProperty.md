---
description: Architectural reference for ConstantNoiseProperty
---

# ConstantNoiseProperty

**Package:** com.hypixel.hytale.server.worldgen.util
**Type:** Utility

## Definition
```java
// Signature
public final class ConstantNoiseProperty {
```

## Architecture & Concepts
The ConstantNoiseProperty class is a static constant provider, designed to supply shared, immutable instances of NoiseProperty for the world generation system. Its primary role is to serve as a centralized, high-performance source for noise functions that always return a fixed value, specifically 0.0 or 1.0.

This utility prevents the repeated allocation and initialization of new ConstantNoise and SingleNoiseProperty objects throughout the procedural generation codebase. By providing global singletons, it reduces memory overhead and garbage collection pressure during computationally intensive world generation tasks.

Architecturally, it is a foundational primitive within the procedural library. It is frequently used as a default value, a fallback, or a base layer in more complex noise compositions. For example, it can define a region with a uniform characteristic, such as a perfectly flat water level (using DEFAULT_ZERO) or a zone of maximum material density (using DEFAULT_ONE).

### Lifecycle & Ownership
- **Creation:** The public static fields, DEFAULT_ZERO and DEFAULT_ONE, are instantiated by the Java Virtual Machine during static initialization. This occurs exactly once when the ConstantNoiseProperty class is first loaded into memory.
- **Scope:** The provided NoiseProperty instances are application-scoped. They persist for the entire lifetime of the server process.
- **Destruction:** The objects are eligible for garbage collection only upon JVM shutdown. There is no manual destruction mechanism.

## Internal State & Concurrency
- **State:** This class is stateless. The public fields it exposes are final and reference objects that are themselves deeply immutable. The underlying noise values are hardcoded and cannot be modified at runtime.
- **Thread Safety:** The class is unconditionally thread-safe. As a provider of immutable singletons, its fields can be accessed and used by any number of concurrent world generation threads without requiring any synchronization or locking mechanisms. This design is critical for enabling safe, parallelized chunk generation.

## API Surface
The public contract of this class consists solely of static fields. It has no methods and cannot be instantiated.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DEFAULT_ZERO | NoiseProperty | O(1) | Provides a shared, immutable NoiseProperty that always evaluates to 0.0. |
| DEFAULT_ONE | NoiseProperty | O(1) | Provides a shared, immutable NoiseProperty that always evaluates to 1.0. |

## Integration Patterns

### Standard Usage
The intended use is to directly reference the static fields where a constant noise value is required within a world generation algorithm. This avoids object allocation and ensures consistent behavior.

```java
// Example: Setting a biome's base height to a flat zero
import com.hypixel.hytale.server.worldgen.util.ConstantNoiseProperty;
import com.hypixel.hytale.procedurallib.property.NoiseProperty;

// ... inside a world generator configuration ...
NoiseProperty terrainHeight = ConstantNoiseProperty.DEFAULT_ZERO;
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The class has a private constructor that throws an UnsupportedOperationException. Attempting to create an instance via `new ConstantNoiseProperty()` will result in a runtime error. This is an intentional design choice to enforce its role as a static utility.
- **Reflection-based Modification:** Attempting to use reflection to modify the final fields of this class or the internal state of the referenced NoiseProperty objects is a severe violation of its contract. Doing so would introduce undefined behavior across the entire world generation system, as all modules expect these values to be constant.

## Data Pipeline
This class does not process data; it is a source of constant data for other systems. It typically sits at the beginning of a procedural generation pipeline.

> Flow:
> **ConstantNoiseProperty.DEFAULT_ZERO** -> Noise Composition Algorithm -> Voxel Data Generation -> Chunk Serialization

