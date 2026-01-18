---
description: Architectural reference for BodyMotionWander
---

# BodyMotionWander

**Package:** com.hypixel.hytale.server.npc.corecomponents.movement
**Type:** Component / Transient

## Definition
```java
// Signature
public class BodyMotionWander extends BodyMotionWanderBase {
```

## Architecture & Concepts
The BodyMotionWander class is a concrete implementation of a movement strategy for server-side Non-Player Characters (NPCs). It represents the most basic form of wandering behavior, where an NPC moves towards a generated point without any special environmental constraints.

This component fits into the broader NPC Behavior and Movement system. The engine treats various movement types polymorphically through a common base, likely `BodyMotion`. This class, inheriting from `BodyMotionWanderBase`, provides a specific implementation for the `constrainMove` hook method. Its primary architectural role is to serve as a default, uninhibited wandering module that can be attached to an NPC's component set. The core logic for selecting a wander destination resides in its parent, `BodyMotionWanderBase`; this class merely confirms that the proposed movement is always valid.

## Lifecycle & Ownership
- **Creation:** An instance of BodyMotionWander is created by the server's asset loading pipeline via its corresponding builder, `BuilderBodyMotionWander`. This occurs when an NPC definition that specifies this movement type is parsed and instantiated. It is not intended for manual creation.
- **Scope:** The lifecycle of a BodyMotionWander instance is strictly tied to the lifecycle of the NPC entity it is attached to. It persists as long as the NPC exists within the game world.
- **Destruction:** The object is eligible for garbage collection when its parent NPC entity is removed from the `EntityStore` and dereferenced. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** This specific class is stateless. All mutable state, such as the current wander target or internal timers, is managed by its parent class, `BodyMotionWanderBase`.
- **Thread Safety:** **WARNING:** This component is not thread-safe. It is designed to be accessed and manipulated exclusively by the main server tick thread for the world region in which the NPC resides. Unsynchronized access from other threads will lead to race conditions and unpredictable entity behavior.

## API Surface
The public API is limited to the constructor, which is only intended for use by the builder system. The primary functional contract is the protected `constrainMove` method, which is called by the engine's movement subsystem.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| constrainMove(...) | double | O(1) | Overrides the base class to apply movement constraints. This implementation applies no constraints, returning the requested move distance unmodified. |

## Integration Patterns

### Standard Usage
A game designer or developer does not interact with this class directly in code. Instead, it is specified declaratively within an NPC's asset definition file (e.g., JSON or HOCON). The engine's asset pipeline is responsible for instantiation.

```yaml
# Example NPC Asset Definition (Conceptual)
components: [
  {
    type: "BodyMotionWander",
    # Parameters for BodyMotionWanderBase would be defined here
    wanderRadius: 10.0,
    idleTime: 5.0
  }
]
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BodyMotionWander()`. The component requires a fully initialized context from the `BuilderSupport` and its specific `BuilderBodyMotionWander`, which is only available during the engine's standard NPC creation process.
- **State Manipulation:** Do not attempt to cast this component to its base class to manipulate its internal state directly. Behavior should be influenced through the appropriate AI and Role systems.

## Data Pipeline
This component acts as a simple passthrough node in the NPC movement calculation pipeline.

> Flow:
> NPC Behavior Tree -> Selects "Wander" Goal -> `BodyMotionWanderBase` calculates target & move distance -> Engine Movement Subsystem -> Calls `constrainMove` on **`BodyMotionWander`** -> Returns unmodified move distance -> Engine updates NPC position in `EntityStore`

---

