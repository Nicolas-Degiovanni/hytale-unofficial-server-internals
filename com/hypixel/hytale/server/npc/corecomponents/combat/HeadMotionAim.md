---
description: Architectural reference for HeadMotionAim
---

# HeadMotionAim

**Package:** com.hypixel.hytale.server.npc.corecomponents.combat
**Type:** Transient

## Definition
```java
// Signature
public class HeadMotionAim extends HeadMotionBase {
```

## Architecture & Concepts

The HeadMotionAim class is a specialized implementation of the HeadMotionBase component, designed to solve the complex problem of aiming an NPC's head at a target for combat. It functions as a high-level "solver" within the NPC AI framework, translating sensory data about a target into precise motor control outputs.

Its primary architectural role is to decouple aiming logic from the NPC's core behavior tree or state machine. Instead of an NPC's behavior directly calculating angles, it delegates this responsibility to HeadMotionAim. This component encapsulates the mathematical complexity of both simple line-of-sight aiming and advanced ballistic trajectory prediction, including target leading (deflection).

Configuration is managed through a Builder pattern (BuilderHeadMotionAim), allowing designers to define distinct aiming profiles for different NPC types. An expert archer, for example, would be configured with low spread and high hit probability, while a less skilled combatant might have a high spread value, resulting in less accurate attacks.

## Lifecycle & Ownership

-   **Creation:** HeadMotionAim instances are not created directly via their constructor in game logic. They are instantiated by the server's NPC asset loading system, using a corresponding BuilderHeadMotionAim. This occurs when an NPC's behavior profile is loaded and its components are assembled.
-   **Scope:** The lifetime of a HeadMotionAim object is tightly coupled to the NPC entity that owns it, and more specifically, to the active combat Role or behavior that requires aiming. It is a stateful, per-NPC instance.
-   **Destruction:** The object is eligible for garbage collection when its owning NPC is unloaded or when its parent Role is deactivated and replaced. There is no explicit destruction or cleanup method; its lifecycle is managed entirely by the NPC component system.

## Internal State & Concurrency

-   **State:** This class is highly stateful and mutable. It maintains significant internal state through the AimingData object, which caches ballistic solutions, target references, and attack status. It also tracks the last calculated random spread across ticks. This state is critical for multi-tick aiming calculations and for applying inaccuracy penalties only once per attack sequence.

-   **Thread Safety:** **This component is not thread-safe and must not be accessed concurrently.** It is designed to be exclusively owned and operated by a single NPC's update logic within the server's main game loop. All method calls are expected to be serialized per-instance.

    **Warning:** Sharing a HeadMotionAim instance between multiple NPCs will lead to catastrophic state corruption and unpredictable behavior. The use of ThreadLocalRandom indicates an expectation of single-threaded access within a potentially parallelized tick system (where each NPC update may run in a separate context).

## API Surface

The public contract is focused on integrating with the NPC's main update loop.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| preComputeSteering(...) | void | O(1) | Pre-tick setup phase. Primarily used to pass the internal AimingData state to other components via the InfoProvider. |
| activate(...) | void | O(1) | Signals the start of a new attack sequence. Resets the internal state to allow for a new spread calculation. |
| computeSteering(...) | boolean | O(N) | The core operational method. Calculates the desired yaw and pitch. Complexity depends on whether a ballistic solution is required. Mutates the provided Steering object with the result. |

## Integration Patterns

### Standard Usage

HeadMotionAim is driven by a higher-level system, typically an NPC Role. The Role gathers sensory information, invokes the component to calculate aiming, and applies the resulting steering to the NPC's motor control.

```java
// Executed within an NPC's Role or Behavior update method

// 1. Obtain sensory information about the target
InfoProvider sensorInfo = npc.getSensorSystem().findBestTarget();

// 2. Get the component instance and an output object
HeadMotionAim aimer = npc.getComponent(HeadMotionAim.class);
Steering desiredSteering = new Steering();

// 3. Activate to signal a new shot, resetting spread calculation
if (shouldFireThisTick) {
    aimer.activate(npc.getRef(), this, componentAccessor);
}

// 4. Delegate the aiming calculation to the component
aimer.computeSteering(
    npc.getRef(),
    this, // The current Role
    sensorInfo,
    deltaTime,
    desiredSteering,
    componentAccessor
);

// 5. The desiredSteering object now contains the target yaw and pitch
//    and is passed to the NPC's motor system.
npc.getMotorSystem().applySteering(desiredSteering);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new HeadMotionAim()`. This bypasses the critical configuration applied by the builder system, resulting in an unconfigured component with default (and likely incorrect) aiming parameters.
-   **State Mismanagement:** Calling computeSteering repeatedly without calling activate between distinct attacks will prevent the aiming spread from being recalculated, causing the NPC to miss in a predictable pattern rather than a random one.
-   **Instance Sharing:** Do not cache and reuse a single HeadMotionAim instance for multiple NPCs. Its internal state (lastTargetReference, spreadX/Y/Z) is specific to one NPC's aiming context.

## Data Pipeline

HeadMotionAim acts as a transformation stage in the NPC's perception-to-action data pipeline.

> Flow:
> NPC Sensor System -> InfoProvider (Target Data) -> **HeadMotionAim** (Consumes Target Data, computes angles) -> Steering (Output Object) -> NPC Motor System -> HeadRotation (Final Component Update)

