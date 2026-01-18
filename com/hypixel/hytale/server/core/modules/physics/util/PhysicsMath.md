---
description: Architectural reference for PhysicsMath
---

# PhysicsMath

**Package:** com.hypixel.hytale.server.core.modules.physics.util
**Type:** Utility

## Definition
```java
// Signature
public class PhysicsMath {
```

## Architecture & Concepts
The PhysicsMath class is a stateless, static utility class that serves as a foundational mathematics library for the server-side physics engine. It centralizes a collection of pure functions for performing common and complex physics, trigonometric, and geometric calculations.

This class is not a system or a service; it is a low-level toolkit. Its primary architectural role is to provide a single, consistent, and optimized source of truth for physics formulas. By encapsulating these calculations, higher-level systems such as the Entity-Component System (ECS) physics processors or movement controllers can remain focused on logic rather than mathematical implementation. It decouples the physics simulation logic from the underlying mathematical formulas, promoting code reuse and simplifying maintenance.

Key conceptual domains covered include:
*   **Aerodynamics:** Terminal velocity, acceleration based on speed, and drag coefficients.
*   **Fluid Dynamics:** Constants for air and water density.
*   **Collision Primitives:** Calculating intersection volumes and projected areas for bounding boxes.
*   **Vector Kinematics:** Converting between angular representations (heading, pitch) and directional vectors.

## Lifecycle & Ownership
As a static utility class, PhysicsMath does not follow a traditional object lifecycle. It is never instantiated and therefore has no instance-level ownership.

*   **Creation:** The class is loaded into the JVM by the ClassLoader when it is first referenced by any part of the server code, typically during the initialization of the core physics module. No instance is ever created.
*   **Scope:** The class and its static methods are available globally for the entire duration of the server's runtime.
*   **Destruction:** The class is unloaded from memory only when the Java Virtual Machine shuts down. There is no concept of manual destruction or resource cleanup.

## Internal State & Concurrency
*   **State:** PhysicsMath is **completely stateless**. It contains no mutable fields and all its methods operate exclusively on the arguments provided to them. The class fields are all static final constants, making them immutable.
*   **Thread Safety:** This class is **inherently thread-safe**. Due to its stateless and purely functional nature, all methods can be called concurrently from any number of threads without any risk of race conditions or side effects. No synchronization or locking mechanisms are required.

## API Surface
The API consists entirely of static methods that perform a single, well-defined calculation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAcceleration(speed, terminalSpeed) | double | O(1) | Calculates gravitational acceleration adjusted for air resistance, approaching zero as speed nears terminal velocity. |
| getTerminalVelocity(...) | double | O(1) | Computes the terminal velocity of an object based on its mass, density, area, and drag. |
| computeProjectedArea(direction, box) | double | O(1) | Calculates the 2D cross-sectional area of a bounding box as projected onto a plane perpendicular to the given direction vector. |
| volumeOfIntersection(a, posA, b, posB) | double | O(1) | Determines the volume of the overlapping region between two world-aligned bounding boxes. Returns 0 if they do not intersect. |
| vectorFromAngles(heading, pitch, outDirection) | Vector3d | O(1) | Converts heading and pitch angles into a normalized 3D direction vector. The result is written to the provided *outDirection* vector to prevent object allocation. |
| headingFromDirection(x, z) | float | O(1) | Extracts the horizontal heading angle (in radians) from the X and Z components of a direction vector. |

## Integration Patterns

### Standard Usage
PhysicsMath methods should be called statically from any system that needs to perform a physics-related calculation. It is most commonly used within the main server tick loop by systems responsible for entity movement and collision.

```java
// Example: Calculating buoyancy force on an entity
Box entityBounds = entity.getBoundingBox();
Vector3d entityPosition = entity.getPosition();

// A conceptual water volume in the world
Box waterVolume = new Box(...);
Vector3d waterPosition = new Vector3d(...);

double submergedVolume = PhysicsMath.volumeOfIntersection(entityBounds, entityPosition, waterVolume, waterPosition);

if (submergedVolume > 0) {
    // F = p * V * g (Archimedes' principle)
    double buoyancyForce = PhysicsMath.DENSITY_WATER * submergedVolume * WORLD_GRAVITY;
    entity.applyForce(new Vector3d(0, buoyancyForce, 0));
}
```

### Anti-Patterns (Do NOT do this)
*   **Direct Instantiation:** Never create an instance of this class. While the default constructor is public, instantiating it serves no purpose and is misleading. All access must be static.
    ```java
    // ANTI-PATTERN: Pointless object creation
    PhysicsMath mathInstance = new PhysicsMath();
    double vol = mathInstance.volumeOfIntersection(...);
    ```
*   **Redundant Calculation:** Avoid calling these functions with the same inputs multiple times within the same tick for the same entity. Cache the results for the duration of the tick if they are needed in multiple places.

## Data Pipeline
PhysicsMath does not participate in a data pipeline as a processing stage. Instead, it acts as a stateless computational tool invoked by other systems *within* their own data processing pipelines. A typical integration is within the server's physics update loop.

> Flow:
> Entity State (Position, Velocity, BoundingBox) -> Physics Simulation Tick -> **PhysicsMath** (Performs calculations like drag, intersection, etc.) -> Force Accumulator -> New Entity State (Updated Position, Velocity) -> World State Update

