---
description: Architectural reference for CellType
---

# CellType

**Package:** com.hypixel.hytale.procedurallib.logic.cell
**Type:** Type-Safe Enumeration / Data Type

## Definition
```java
// Signature
public enum CellType {
```

## Architecture & Concepts
CellType is a foundational enumeration that defines the fundamental geometric primitive for grid-based procedural generation. It serves as a type-safe discriminator, allowing algorithms within the procedural library to operate on different grid topologies without ambiguity.

This enumeration is a core building block, not a complex service. Its primary architectural role is to enforce a strict contract for grid-related logic. By using CellType instead of string literals or integer constants, the system prevents a class of common errors and makes the intent of grid-processing algorithms explicit. It is consumed by higher-level systems like ZoneGenerators and Pathfinders to select the appropriate mathematical models for distance calculation, neighbor finding, and area filling.

## Lifecycle & Ownership
- **Creation:** Instances of SQUARE and HEX are created and initialized by the Java Virtual Machine during class loading. They are compile-time constants.
- **Scope:** CellType instances are static and persist for the entire lifetime of the application. They are globally accessible.
- **Destruction:** The instances are reclaimed by the garbage collector only when the application's class loader is unloaded, which typically occurs at shutdown.

## Internal State & Concurrency
- **State:** Inherently immutable. The state of an enum constant is fixed at compile time and cannot be altered at runtime.
- **Thread Safety:** CellType is unconditionally thread-safe. As a language-level construct, the JVM guarantees that its instances can be safely accessed from any thread without synchronization.

## Integration Patterns

### Standard Usage
CellType is primarily used as a parameter in method signatures or as a field in configuration objects to control the behavior of grid-based algorithms. It is most effective when used in switch statements to branch logic based on the grid topology.

```java
// A generator configured to operate on a hexagonal grid
GridConfiguration config = new GridConfiguration();
config.setCellType(CellType.HEX);

// Logic branches based on the configured cell type
switch (config.getCellType()) {
    case SQUARE:
        // Execute square-based neighbor finding logic
        break;
    case HEX:
        // Execute hex-based neighbor finding logic
        break;
}
```

### Anti-Patterns (Do NOT do this)
- **Ordinal Comparison:** Do not rely on the `ordinal()` method for logical comparisons. The integer value can change if the declaration order of the enum constants is modified, leading to subtle and dangerous bugs. Always compare instances directly.
- **Instantiation:** Enum types cannot be instantiated using the `new` keyword. Attempting to do so will result in a compile-time error.

## Data Pipeline
CellType is not a processing stage in a data pipeline; rather, it is a critical piece of metadata that directs the flow of data *within* a pipeline. It acts as a discriminator that determines which algorithmic path is taken.

> Flow:
> Generator Configuration -> **CellType** (as a field) -> Grid Processor -> (Switches to Hex or Square logic) -> Final Grid Data

