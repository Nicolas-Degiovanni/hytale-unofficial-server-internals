---
description: Architectural reference for NPCPhysicsMath
---

# NPCPhysicsMath

**Package:** com.hypixel.hytale.server.npc.util
**Type:** Utility

## Definition
```java
// Signature
public class NPCPhysicsMath {
```

## Architecture & Concepts

NPCPhysicsMath is a static, stateless utility class that serves as the foundational mathematics library for the server-side Non-Player Character (NPC) subsystem. It centralizes a wide range of complex and frequently used geometric, physics, and vector calculations required for NPC behavior, movement, and environmental interaction.

The primary architectural role of this class is to decouple high-level AI logic (such as Behavior Trees, Pathfinding, and Combat AI) from the low-level mathematical implementations. By providing a stable, optimized, and globally accessible set of functions, it ensures consistency and correctness across all NPC behaviors, preventing code duplication and subtle physics-related bugs.

This class acts as a bridge between abstract mathematical concepts and concrete game-world data. It contains methods that operate not only on pure vectors but also on game-specific constructs like World, BlockType, and BoundingBox, making it an essential service layer for any system that needs to reason about an NPC's physical presence and capabilities within the game world.

## Lifecycle & Ownership

-   **Creation:** The NPCPhysicsMath class is never instantiated. Its private constructor prevents the creation of objects, enforcing its role as a purely static utility. The class is loaded into the JVM by the class loader upon its first use.
-   **Scope:** As a static class, its methods and constants are available globally for the entire lifetime of the server process.
-   **Destruction:** The class is unloaded from memory only when the JVM shuts down. There is no instance-level state to manage or clean up.

## Internal State & Concurrency

-   **State:** This class is completely **stateless and immutable**. It contains no mutable static fields. All methods are pure functions whose output depends exclusively on their input arguments. Any Vector3d arguments passed for storing results are mutated by design, but the class itself holds no state between calls.

-   **Thread Safety:** NPCPhysicsMath is inherently **thread-safe**. Due to its stateless nature, methods can be called concurrently from multiple threads (e.g., different world simulation threads or AI processing threads) without any risk of race conditions or data corruption. No synchronization primitives, such as locks, are used or required.

## API Surface

The API provides a comprehensive toolkit for 3D spatial reasoning and physics simulation. Below is a representative selection of its key functionalities.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isInViewCone(viewer, viewDir, cosCone, object) | boolean | O(1) | Determines if a target object is within the 3D view cone of a viewer. Critical for perception and targeting systems. |
| intersectSweptSpheres(p1, v1, p2, v2, radius, ...) | int | O(1) | Calculates the time of collision between two moving spheres. Essential for predictive collision detection and avoidance. |
| heightOverGround(world, x, z) | double | O(1) | Queries the world to find the y-coordinate of the ground at a given x,z position, accounting for block shapes. |
| jumpParameters(pos, target, gravity, velocity) | double | O(1) | Calculates the required initial velocity vector for a parabolic jump to a target position under a given gravity. |
| accelerateToTargetSpeed(current, target, dt, ...) | double | O(1) | Models realistic acceleration and deceleration with drag, moving a current velocity towards a target velocity over a time delta. |
| blockEmptySpace(blockType, rotation, direction) | double | O(1) | Calculates the amount of empty space within a block's bounds along a specific axis. Used for fine-grained pathfinding and collision checks. |

## Integration Patterns

### Standard Usage

NPCPhysicsMath is designed for direct, static invocation from any server-side system that manages NPC logic. It is most commonly used within AI behavior nodes, pathfinding algorithms, and physics update ticks.

```java
// Example: An NPC's targeting logic checks if a player is visible and in range.
Vector3d npcPosition = npc.getPosition();
Vector3d npcViewDirection = npc.getViewDirection();
Vector3d playerPosition = player.getPosition();

// Use a cosine of a 45-degree half-angle for the view cone.
float cosViewConeHalfAngle = 0.707f; 

boolean isVisible = NPCPhysicsMath.isInViewCone(
    npcPosition,
    npcViewDirection,
    cosViewConeHalfAngle,
    playerPosition
);

if (isVisible) {
    // The player is in the NPC's field of view.
    // Further logic for engagement can proceed.
}
```

### Anti-Patterns (Do NOT do this)

-   **Attempted Instantiation:** Do not attempt to create an instance of this class. The private constructor will result in a compile-time error. All access must be static: `NPCPhysicsMath.method()`.
-   **Ignoring Floating-Point Precision:** Do not use direct equality checks (`==`) on the results of physics calculations. The class provides `near()` methods for safely comparing floating-point numbers and vectors within a small epsilon. Failure to do so can lead to unreliable and buggy behavior.
-   **Misusing Output Parameters:** Many methods accept a `Vector3d result` parameter to avoid object allocation. The caller is responsible for providing a valid, non-null vector. Assuming the method returns a new instance will lead to NullPointerExceptions.

## Data Pipeline

NPCPhysicsMath does not operate as a data pipeline itself but rather as a computational service within larger pipelines, such as the NPC update loop. It is a pure function library that takes in data, performs a calculation, and returns a result without side effects beyond mutating designated output parameters.

> **Flow: NPC Behavior System to Physics Calculation**
>
> 1.  An **AI Behavior Node** (e.g., "ChaseTargetNode") retrieves current state data, such as the NPC's position, velocity, and the target's position.
> 2.  The node invokes a static method on **NPCPhysicsMath**, passing the state data as arguments. For example, `accelerateToTargetSpeed(...)`.
> 3.  **NPCPhysicsMath** executes the mathematical computation in a stateless manner.
> 4.  The calculated result (e.g., a new velocity value) is returned to the Behavior Node.
> 5.  The Behavior Node applies this result to the NPC's **Physics Component**, which will be processed by the main server physics simulation tick.

