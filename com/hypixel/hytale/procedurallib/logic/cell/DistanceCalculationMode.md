---
description: Architectural reference for DistanceCalculationMode
---

# DistanceCalculationMode

**Package:** com.hypixel.hytale.procedurallib.logic.cell
**Type:** Utility (Enum)

## Definition
```java
// Signature
public enum DistanceCalculationMode {
```

## Architecture & Concepts
The DistanceCalculationMode enum is a core component of the procedural generation library, implementing the **Strategy Pattern** to define interchangeable algorithms for calculating distance. Its primary role is to decouple high-level procedural logic, such as Voronoi diagram generation or noise field evaluation, from the specific mathematical formula used to measure proximity between points.

Each enum constant (EUCLIDEAN, MANHATTAN, etc.) encapsulates a distinct distance metric within a PointDistanceFunction instance. This allows a single procedural generator to produce vastly different structural patterns simply by being configured with a different DistanceCalculationMode, without any changes to the generator's own code. This provides significant flexibility for world generation artists and designers.

For example, EUCLIDEAN distance produces the familiar circular or spherical patterns, while MANHATTAN distance produces square or diamond-shaped patterns, and MAX distance produces perfectly square patterns. These are fundamental building blocks for creating varied and stylized procedural content.

## Lifecycle & Ownership
- **Creation:** Enum constants are instantiated automatically and exclusively by the Java Virtual Machine (JVM) during class loading. There is only one instance of each constant (e.g., EUCLIDEAN) for the entire application lifetime.
- **Scope:** Application-wide static constants. They are available as soon as the DistanceCalculationMode class is loaded and persist until the application terminates.
- **Destruction:** Managed entirely by the JVM. The constants are eligible for garbage collection only when the defining class loader is unloaded, which typically occurs at application shutdown. Developers have no control over this process.

## Internal State & Concurrency
- **State:** This class is **deeply immutable**. Each enum constant holds a final reference to a stateless PointDistanceFunction implementation. The distance calculation methods are pure functions; their output depends solely on their input arguments, with no side effects or reliance on external state.
- **Thread Safety:** This class is **inherently thread-safe**. Due to its immutable and stateless nature, instances of DistanceCalculationMode can be safely shared and used across any number of threads without requiring locks or any other synchronization mechanisms.

## API Surface
The public contract is minimal, focusing on providing access to the underlying calculation strategy.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getFunction() | PointDistanceFunction | O(1) | Retrieves the encapsulated distance calculation strategy object. |
| from(function) | static DistanceCalculationMode | O(N) | Performs a reverse lookup to find the enum constant matching a given function instance. Returns null if no match is found. Complexity is constant-time as N is fixed and small. |

## Integration Patterns

### Standard Usage
Systems requiring distance calculations should accept a DistanceCalculationMode as a configuration parameter. The caller selects the desired mode and passes it to the system, which then uses getFunction to retrieve and execute the appropriate algorithm.

```java
// A procedural system is configured with a specific distance mode.
VoronoiGenerator generator = new VoronoiGenerator();
generator.setDistanceMode(DistanceCalculationMode.MANHATTAN);

// The generator internally uses the function to perform its calculations.
PointDistanceFunction distanceFunc = generator.getDistanceMode().getFunction();
double distance = distanceFunc.distance2D(dx, dy);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Enums cannot be instantiated via a constructor. Attempting to use `new DistanceCalculationMode()` will result in a compile-time error.
- **Reliance on Ordinals:** Do not use the `ordinal()` method to serialize or identify a mode. If the declaration order of the enum constants changes in the future, all saved data and logic will break. Use the `name()` method for stable serialization.
- **Null Function Lookups:** The static `from` method returns null if a matching function is not found. Callers must perform a null check to avoid a NullPointerException.

## Data Pipeline
This class does not process a stream of data. Instead, it acts as a factory or provider of strategy objects that are injected into other data-processing systems.

> Flow:
> System Configuration -> **DistanceCalculationMode** -> getFunction() -> Procedural Algorithm (e.g., Voronoi Generator) -> Calculated Distance Value

