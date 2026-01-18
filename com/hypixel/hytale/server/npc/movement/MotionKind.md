---
description: Architectural reference for MotionKind
---

# MotionKind

**Package:** com.hypixel.hytale.server.npc.movement
**Type:** Static Data Model / Enum

## Definition
```java
// Signature
public enum MotionKind {
```

## Architecture & Concepts
MotionKind is a foundational enumeration that defines the discrete, high-level states of an NPC's physical movement. It serves as a shared vocabulary between the server-side physics simulation, the AI behavior tree, and the animation system.

This enum is not a system with logic, but rather a critical data model that represents the *output* of the physics engine's analysis of an NPC's velocity, position, and environmental context (e.g., in water, in air). By categorizing complex physics data into these simple, well-defined states, other systems can react without needing to interpret raw physics vectors. For example, the animation system can subscribe to changes in MotionKind to trigger the correct animation (e.g., switch from a run cycle to a swim cycle).

It forms the core of a finite state machine for NPC physical expression.

## Lifecycle & Ownership
- **Creation:** The Java Virtual Machine creates and initializes all enum instances when the MotionKind class is first loaded by the ClassLoader. This process is guaranteed to happen only once.
- **Scope:** As static final constants, all MotionKind instances (ASCENDING, STANDING, etc.) exist for the entire lifetime of the application. They are globally accessible and effectively permanent.
- **Destruction:** Instances are reclaimed only when the ClassLoader that loaded them is garbage collected, which typically only occurs upon application shutdown.

## Internal State & Concurrency
- **State:** Inherently immutable. The state of an enum constant cannot be changed after its creation during class loading.
- **Thread Safety:** MotionKind is unconditionally thread-safe. As immutable singletons managed by the JVM, its constants can be safely read and passed between any number of threads without synchronization. This is a critical guarantee for the multithreaded server environment.

## API Surface
The primary API is the set of defined constants themselves.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ASCENDING | MotionKind | O(1) | The NPC is moving upwards, typically by climbing a ladder or vine. |
| DESCENDING | MotionKind | O(1) | The NPC is moving downwards, typically by climbing a ladder or vine. |
| DROPPING | MotionKind | O(1) | The NPC is in freefall. Gravity is the primary force acting upon it. |
| STANDING | MotionKind | O(1) | The NPC has negligible horizontal velocity and is on solid ground. |
| MOVING | MotionKind | O(1) | The NPC has significant horizontal velocity and is on solid ground. |
| FLYING | MotionKind | O(1) | The NPC is airborne but not in freefall; it is under self-propulsion. |
| SWIMMING | MotionKind | O(1) | The NPC is in a liquid and has horizontal velocity. |
| SWIMMING_TURNING | MotionKind | O(1) | The NPC is in a liquid and is changing direction, but not necessarily moving forward. |

## Integration Patterns

### Standard Usage
MotionKind is most commonly used within conditional logic, such as switch statements, to execute different behaviors based on the NPC's current physical state.

```java
// How a developer should normally use this
NPCMovementComponent movement = npc.get(NPCMovementComponent.class);
MotionKind currentMotion = movement.getMotionKind();

switch (currentMotion) {
    case STANDING:
        playIdleAnimation();
        break;
    case MOVING:
        playRunCycle();
        break;
    case DROPPING:
        playFallingAnimation();
        break;
    // ... other cases
}
```

### Anti-Patterns (Do NOT do this)
- **Ordinal Comparison:** Do not rely on the `ordinal()` method for comparisons or logic. The integer value is fragile and will change if the declaration order of the enum constants is modified, leading to subtle and severe bugs. Always compare instances directly.
- **BAD:** `if (motion.ordinal() == 3)`
- **GOOD:** `if (motion == MotionKind.STANDING)`
- **String Comparison:** Do not use `toString()` or `name()` for logical comparisons. This is inefficient and less safe than direct object comparison.
- **BAD:** `if (motion.name().equals("STANDING"))`
- **GOOD:** `if (motion == MotionKind.STANDING)`

## Data Pipeline
MotionKind acts as a translation point, converting low-level physics data into a high-level, semantic state.

> Flow:
> NPC Physics Update -> Velocity & Collision Analysis -> **MotionKind Determination** -> NPCMovementComponent State Update -> AI Behavior Tree & Animation System Reaction

