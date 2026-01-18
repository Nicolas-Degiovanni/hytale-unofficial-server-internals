---
description: Architectural reference for AimingData
---

# AimingData

**Package:** com.hypixel.hytale.server.npc.util
**Type:** Transient Component

## Definition
```java
// Signature
public class AimingData implements ExtraInfoProvider {
```

## Architecture & Concepts

The AimingData class is a stateful computational component responsible for calculating the required pitch and yaw for a server-side NPC to hit a target. It is not a singleton service but rather a data-holding object that encapsulates the complex mathematics of trajectory planning. It functions as a bridge between an NPC's sensory input (target position and velocity) and its motor output (orientation and attack execution).

The component supports two distinct aiming paradigms:
1.  **Direct Aiming:** A simple line-of-sight vector calculation used for melee or hitscan attacks. This is activated via the `requireCloseCombat` method.
2.  **Ballistic Aiming:** A sophisticated trajectory calculation for projectiles, accounting for gravity, muzzle velocity, and target motion. This mode is activated by `requireBallistic` and relies on solving a quartic equation to determine the optimal launch angles.

A key architectural feature is its ability to compute and store up to two valid ballistic solutions: a low-angle (flat) trajectory and a high-angle (lofted) trajectory. This allows the parent AI system to select the most tactically appropriate firing solution. The internal ownership model, managed by `tryClaim` and `release`, ensures that a single, authoritative AI behavior controls the NPC's aiming at any given time, preventing conflicts between different behaviors (e.g., a "ranged attack" behavior and a "look at player" behavior).

## Lifecycle & Ownership

-   **Creation:** An AimingData instance is typically created and owned by a higher-level NPC component or AI behavior. Its implementation of the ExtraInfoProvider interface suggests it is designed to be attached to an NPC's state container and retrieved as needed by various AI systems.
-   **Scope:** The object's state is transient and scoped to a single targeting task. An AI behavior will `tryClaim` the object, use it for the duration of the attack or aiming sequence, and then `release` it. It does not persist across different targets or behaviors without being explicitly cleared.
-   **Destruction:** The object is eligible for garbage collection when its owning NPC or AI component is destroyed. The `clear` and `release` methods are critical for resetting its state, allowing the same instance to be reused for subsequent aiming tasks within the NPC's lifetime, thus reducing object churn.

## Internal State & Concurrency

-   **State:** AimingData is highly mutable. Its primary purpose is to cache the results of the expensive `computeSolution` calculation. Fields like `pitch`, `yaw`, `haveSolution`, and `target` are continuously updated by the owning AI system.
-   **Thread Safety:** This class is **not thread-safe** and is designed for single-threaded access. All state is mutated directly without locks or synchronization primitives. It is expected to be exclusively accessed and modified by the main server thread during an NPC's update tick. The `tryClaim` mechanism provides a logical, cooperative lock to manage state access between different AI behaviors, not between different threads.

**WARNING:** Concurrent modification from multiple threads will lead to race conditions, corrupted aiming solutions, and unpredictable server behavior. All interactions must be synchronized with the server's main game loop.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| requireBallistic(BallisticData data) | void | O(1) | Configures the component for projectile aiming and resets any existing solution. |
| requireCloseCombat() | void | O(1) | Configures the component for direct line-of-sight aiming and resets any existing solution. |
| computeSolution(x, y, z, vx, vy, vz) | boolean | O(N) | Computes the aiming solution. Complexity is high for ballistic mode due to quartic root-solving. Returns true if a valid solution was found. |
| getPitch() / getYaw() | float | O(1) | Retrieves the calculated pitch or yaw for the currently selected trajectory type. |
| setTarget(Ref<EntityStore> ref) | void | O(1) | Sets the entity target for the aiming calculation. |
| tryClaim(int id) | void | O(1) | Attempts to claim ownership of the component for a specific AI behavior, identified by an integer ID. |
| release() | void | O(1) | Releases ownership, allowing another system to claim it. |
| clear() | void | O(1) | Resets all internal state, including the aiming solution, target, and configuration. |

## Integration Patterns

### Standard Usage

An NPC's AI behavior (e.g., an AttackTask) is the primary consumer. The standard lifecycle involves claiming the component, configuring it for the desired attack type, and repeatedly computing the solution each tick until the NPC is on target.

```java
// Within an NPC's AI update tick
AimingData aimingData = npc.getInfo(AimingData.class);

// Attempt to take control for this attack behavior
aimingData.tryClaim(this.getBehaviorId());
if (!aimingData.isClaimedBy(this.getBehaviorId())) {
    return; // Another behavior is using it
}

// Configure and update the solution
aimingData.requireBallistic(projectileConfig.getBallisticData());
aimingData.setTarget(currentTargetRef);

boolean hasSolution = aimingData.computeSolution(relativeX, relativeY, relativeZ, targetVelX, targetVelY, targetVelZ);

if (hasSolution) {
    // Feed the solution to the NPC's orientation/motor systems
    npc.getOrientation().setLookDirection(aimingData.getYaw(), aimingData.getPitch());
}
```

### Anti-Patterns (Do NOT do this)

-   **Shared Instances:** Do not share a single AimingData instance between multiple NPCs. Its state is intrinsically tied to one owner and one target.
-   **Ignoring Ownership:** Bypassing the `tryClaim` and `release` flow can cause different AI behaviors to fight for control, resulting in erratic aiming as they overwrite each other's calculations.
-   **State Leakage:** Failure to call `clear` or re-configure the component (`requireBallistic`/`requireCloseCombat`) between different attack types can lead to using stale data from a previous task.

## Data Pipeline

AimingData acts as a computational stage in the NPC's targeting pipeline, transforming raw positional data into a usable directional vector.

> Flow:
> NPC Sensor System (Provides Target Position/Velocity) -> AI Behavior (Invokes `computeSolution`) -> **AimingData** (Caches Pitch/Yaw Solution) -> NPC Orientation Component (Applies Rotation) -> Attack Execution System (Launches Projectile)

