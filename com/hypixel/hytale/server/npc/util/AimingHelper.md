---
description: Architectural reference for AimingHelper
---

# AimingHelper

**Package:** com.hypixel.hytale.server.npc.util
**Type:** Utility

## Definition
```java
// Signature
public class AimingHelper {
```

## Architecture & Concepts
The AimingHelper is a stateless, computational utility class designed to solve ballistic trajectory problems for server-side AI. It serves as a foundational component within the Non-Player Character (NPC) combat system, specifically for entities that utilize projectile attacks.

This class encapsulates the complex physics of projectile motion under gravity. By providing a clean, static API, it decouples the low-level mathematical calculations from the high-level AI decision-making logic found in behavior trees or state machines. An NPC's "Attack" behavior does not need to understand parabolic arcs; it only needs to query this helper to determine if a shot is possible and what the required launch angle is.

The core function, computePitch, solves for the launch angle required to hit a target at a given distance and relative height. Critically, it correctly models that two possible solutions (a low arc and a high arc) often exist for the same target, returning both for the calling system to decide upon.

A key architectural decision is the gravity threshold constant, MIN_GRAVITY_FOR_PARABOLA. Below this value, the system approximates the trajectory as a straight line. This is a performance optimization and a stability feature, preventing floating-point errors and erratic behavior in low-gravity or zero-gravity environments.

## Lifecycle & Ownership
- **Creation:** As a static utility class, AimingHelper is never instantiated. Its bytecode is loaded into the JVM by the ClassLoader when it is first referenced by another class, typically an NPC's combat behavior module.
- **Scope:** The class and its static methods are available at the application level for the entire lifetime of the server process.
- **Destruction:** The class is unloaded from the JVM only when the server shuts down and its ClassLoader is garbage collected. There is no concept of instance-level destruction.

## Internal State & Concurrency
- **State:** AimingHelper is **stateless** and therefore immutable. It contains no instance or static fields that store data between calls. All computations are derived exclusively from the arguments provided to its methods.
- **Thread Safety:** This class is inherently **thread-safe**. Its methods are pure functions, meaning they produce no side effects and will always return the same output for the same input. It can be safely invoked from multiple AI update threads concurrently without requiring any locks or synchronization primitives.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ensurePossibleThrowSpeed(distance, y, gravity, throwSpeed) | double | O(1) | Calculates the minimum velocity required to reach a target. Returns the greater of the calculated minimum or the input speed. Prevents AI from attempting physically impossible throws. |
| computePitch(distance, height, velocity, gravity, resultingPitch) | boolean | O(1) | Calculates the required launch pitch(es) to hit a target. Returns false if the target is unreachable. Results are written to the `resultingPitch` output array. |

**WARNING:** The computePitch method requires the `resultingPitch` array to have a length of exactly 2. Failure to provide an array of this size will result in an `IllegalArgumentException`.

## Integration Patterns

### Standard Usage
This helper should be invoked from within an AI behavior node responsible for executing a ranged attack. The standard flow involves querying for a valid pitch and then using the result to orient the NPC before spawning the projectile.

```java
// Within an NPC's attack logic
// 1. Get target data and world physics
Vector3d targetPos = target.getPosition();
Vector3d selfPos = self.getPosition();
double distance = selfPos.distanceHorizontal(targetPos);
double height = targetPos.y - selfPos.y;
double gravity = world.getGravity();
double projectileSpeed = self.getProjectileSpeed();

// 2. Query the AimingHelper
float[] launchAngles = new float[2];
boolean canHit = AimingHelper.computePitch(distance, height, projectileSpeed, gravity, launchAngles);

// 3. Act on the result
if (canHit) {
    // Choose an angle (e.g., the lower, faster arc)
    float chosenPitch = launchAngles[0];
    self.setAimPitch(chosenPitch);
    self.launchProjectile();
} else {
    // Target is out of range, switch to a different behavior
    self.getBehaviorTree().failCurrentTask();
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never attempt to create an instance with `new AimingHelper()`. All methods are static and should be called directly on the class.
- **Ignoring Return Value:** The boolean return from computePitch is critical. Ignoring it and using the contents of the `resultingPitch` array can lead to the NPC using stale or uninitialized data, causing projectiles to fire in erroneous directions when the target is actually unreachable.
- **Assuming a Solution:** Do not assume a target is always hittable. Always check the return value of computePitch before proceeding with an attack.

## Data Pipeline
AimingHelper acts as a pure computational function within the broader AI data flow. It does not transform or pass data through; it generates new data based on world state.

> Flow:
> AI Behavior (Attack State) -> World State (Positions, Gravity) -> **AimingHelper.computePitch** -> Calculated Launch Angle (float) -> Entity Orientation System -> Projectile Spawn System

