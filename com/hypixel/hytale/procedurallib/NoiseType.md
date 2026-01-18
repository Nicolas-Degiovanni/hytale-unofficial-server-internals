---
description: Architectural reference for NoiseType
---

# NoiseType

**Package:** com.hypixel.hytale.procedurallib
**Type:** Static Enumeration

## Definition
```java
// Signature
public enum NoiseType
```

## Architecture & Concepts
The NoiseType enumeration serves as a type-safe discriminator for selecting a specific noise generation algorithm within the procedural generation library. It is a foundational component that provides a clear, self-documenting contract for the available noise functions, such as SIMPLEX, PERLIN, or CELL.

Architecturally, NoiseType decouples the configuration of a procedural system from the implementation of the noise algorithms themselves. By using this enum instead of magic strings or integer constants, the system ensures compile-time safety and prevents runtime errors caused by invalid algorithm identifiers. It is the primary input for factories or services responsible for instantiating concrete noise generator objects.

## Lifecycle & Ownership
- **Creation:** NoiseType constants are instantiated by the Java Virtual Machine (JVM) during class loading. This process is automatic and occurs the first time the NoiseType enum is referenced in the code.
- **Scope:** As a static enumeration, all constants exist for the entire lifetime of the application. They are effectively global, immutable singletons.
- **Destruction:** The enum and its constants are unloaded from memory only when the application's ClassLoader is garbage collected, which typically happens at application shutdown.

## Internal State & Concurrency
- **State:** NoiseType is a stateless and deeply immutable enumeration. Each constant is a fixed value with no mutable fields.
- **Thread Safety:** This enum is inherently thread-safe. Its constants can be safely accessed and passed between any number of threads without synchronization. This is a guaranteed property of Java enumerations.

## API Surface
The public contract of NoiseType consists of its defined constants. These are used for identity comparison and as parameters for other systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SIMPLEX | NoiseType | N/A | Standard Simplex noise algorithm. |
| OLD_SIMPLEX | NoiseType | N/A | A legacy or alternative Simplex implementation. |
| VALUE | NoiseType | N/A | Value noise algorithm. |
| PERLIN | NoiseType | N/A | Classic Perlin noise algorithm. |
| CELL | NoiseType | N/A | Cellular noise, often used for Worley or Voronoi patterns. |
| DISTANCE | NoiseType | N/A | A noise function based on distance fields. |
| CONSTANT | NoiseType | N/A | A generator that returns a constant value. |
| GRID | NoiseType | N/A | A grid-based noise or pattern. |
| MESH | NoiseType | N/A | Noise derived from a mesh structure. |
| BRANCH | NoiseType | N/A | A specialized algorithm for creating branching patterns. |
| POINT | NoiseType | N/A | A noise function based on a set of points. |

## Integration Patterns

### Standard Usage
NoiseType is primarily used to configure a noise generator or within a switch statement to branch logic based on the selected algorithm.

```java
// Example: Selecting a noise generator from a factory
NoiseGenerator generator = NoiseFactory.create(NoiseType.SIMPLEX, seed);
float value = generator.getNoise(x, y);

// Example: Logic branching
switch (selectedNoise) {
    case PERLIN:
        // ... handle Perlin-specific logic
        break;
    case CELL:
        // ... handle Cell-specific logic
        break;
    default:
        throw new UnsupportedOperationException("Noise type not supported");
}
```

### Anti-Patterns (Do NOT do this)
- **Serialization via Ordinal:** Do not persist or transmit the integer ordinal of an enum constant. The order of declaration can change between versions, leading to catastrophic data corruption. Use the string name via the `name()` method for stable serialization.
- **Ordinal-based Logic:** Avoid writing logic that depends on the relative order of the constants, such as `if (type.ordinal() > NoiseType.PERLIN.ordinal())`. This creates brittle code that will break if new types are inserted.

## Data Pipeline
NoiseType acts as a control signal or configuration parameter at the beginning of the noise generation pipeline. It does not process data itself but dictates which processing path will be taken.

> Flow:
> World Generation Configuration -> Noise Function Factory -> **NoiseType** (as parameter) -> Selects Concrete Algorithm -> Noise Value Generation

