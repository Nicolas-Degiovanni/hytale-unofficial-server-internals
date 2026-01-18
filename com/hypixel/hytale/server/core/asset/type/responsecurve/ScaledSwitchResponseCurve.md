---
description: Architectural reference for ScaledSwitchResponseCurve
---

# ScaledSwitchResponseCurve

**Package:** com.hypixel.hytale.server.core.asset.type.responsecurve
**Type:** Model / Data Object

## Definition
```java
// Signature
public class ScaledSwitchResponseCurve extends ScaledResponseCurve {
```

## Architecture & Concepts
The ScaledSwitchResponseCurve is a specialized implementation of the ResponseCurve pattern. It models a discrete, binary step function. Given a continuous input value *x*, it returns one of two predefined constant values: an *initial state* or a *final state*. The transition between these states occurs instantaneously at a configurable *switch point*.

This component is fundamental to the server's data-driven design philosophy. Instead of hard-coding threshold-based logic, behaviors can be defined in external asset files (e.g., JSON) and loaded at runtime. This allows game designers to tune critical gameplay mechanics—such as AI activation ranges, material property changes, or skill effect triggers—without requiring code changes.

Its primary role is to translate a continuous game-state variable (like distance, health percentage, or temperature) into a discrete, binary output that can drive a state machine or a logical branch. The class is defined by its static CODEC field, which dictates the schema for its serialization and deserialization, making it an integral part of the asset pipeline.

## Lifecycle & Ownership
-   **Creation:** Instances are not created directly using the new keyword. The constructor is protected to enforce this. They are exclusively instantiated by the Hytale **Codec** system during the asset loading phase. The static CODEC field is used by a higher-level AssetManager to parse a data definition and construct a fully configured object.
-   **Scope:** The lifetime of a ScaledSwitchResponseCurve instance is bound to the lifecycle of the asset that defines it. It is a stateless, read-only configuration object after creation. It persists as long as its parent asset is held in memory, typically for the duration of a server session or until a world zone is unloaded.
-   **Destruction:** The object is managed by the Java garbage collector. It is eligible for destruction once all references to it are released, which typically occurs when the owning asset is unloaded from the system. There are no manual cleanup or disposal methods.

## Internal State & Concurrency
-   **State:** The internal state consists of three double-precision fields: initialState, finalState, and switchPoint. This state is populated once during deserialization. While the fields are not declared as final, the object should be treated as **effectively immutable** after its construction is complete. Modifying its state at runtime is a severe anti-pattern that can lead to non-deterministic behavior.
-   **Thread Safety:** The class is inherently **thread-safe for all read operations**. The core computeY method is a pure function that only reads from its internal, effectively immutable fields. It can be safely shared and called concurrently by multiple systems (e.g., AI, physics, game rules) without requiring external synchronization or locks.

## API Surface
The public contract is minimal, focusing entirely on its computational purpose.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeY(double x) | double | O(1) | The primary computation method. Returns the configured initialState if x is less than the switchPoint, otherwise returns the finalState. |

## Integration Patterns

### Standard Usage
A ScaledSwitchResponseCurve should be retrieved from a parent configuration or asset. Game logic systems then invoke the computeY method, passing in a relevant real-time value to determine a binary outcome.

```java
// Example: An AI system determining whether to engage a target
// The 'engagementCurve' is loaded from an entity's asset definition.
ResponseCurve engagementCurve = aiComponent.getEngagementCurve();

double distanceToTarget = target.getPosition().distance(self.getPosition());

// computeY returns 0.0 for 'disengaged' and 1.0 for 'engaged'
double engagementState = engagementCurve.computeY(distanceToTarget);

if (engagementState == 1.0) {
    aiComponent.engageTarget();
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never attempt to create an instance with `new ScaledSwitchResponseCurve()`. The protected constructor exists to prevent this. All instances must be created via the asset loading and Codec pipeline to ensure they are correctly configured.
-   **Runtime State Mutation:** Do not use reflection or other means to modify the initialState, finalState, or switchPoint fields after the object has been loaded. This violates its design contract as an immutable configuration object and can cause unpredictable behavior across the server.

## Data Pipeline
The class functions as a terminal node in the asset loading pipeline and as a data source for the game logic pipeline.

> **Asset Loading Flow:**
> Game Asset File (JSON) -> Server AssetManager -> Hytale Codec System -> **ScaledSwitchResponseCurve Instance**

> **Game Logic Flow:**
> Game State Provider (e.g., Physics Engine) -> Input Value (e.g., distance) -> **ScaledSwitchResponseCurve.computeY(x)** -> Binary Output (0.0 or 1.0) -> Decision Logic (e.g., AI State Machine)

