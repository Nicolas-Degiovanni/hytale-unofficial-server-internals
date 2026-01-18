---
description: Architectural reference for the HeadMotion interface, a contract for NPC head-related behaviors.
---

# HeadMotion

**Package:** com.hypixel.hytale.server.npc.instructions
**Type:** Contract Interface

## Definition
```java
// Signature
public interface HeadMotion extends Motion {
}
```

## Architecture & Concepts
The HeadMotion interface is a specialized behavioral contract within the server-side Non-Player Character (NPC) instruction system. It extends the base Motion interface, creating a specific type-safe category for all AI instructions that exclusively control an NPC's head orientation and movement.

This interface serves as a marker and a point of polymorphism. The NPC's core AI processor or behavior tree can query for or dispatch instructions of type HeadMotion without needing to know the concrete implementation (e.g., LookAtTarget, ScanEnvironment, Nod). This decouples the high-level AI logic from the low-level implementation of specific head movements, allowing for modular and extensible NPC behaviors. It is a fundamental component for creating believable and responsive characters.

## Lifecycle & Ownership
As an interface, HeadMotion itself does not have a lifecycle. The following applies to concrete classes that **implement** this interface.

- **Creation:** Instances of HeadMotion implementations are typically created by higher-level AI systems, such as a Behavior Tree, a state machine, or a scripted event handler. They are often short-lived, transient objects created to execute a single, specific action.
- **Scope:** The lifetime of a HeadMotion object is usually tied to the duration of the action it represents. It may exist for a single server tick or persist for several seconds until its goal is achieved or it is interrupted by a higher-priority behavior.
- **Destruction:** The object is eligible for garbage collection once the NPC's instruction processor discards it, either upon completion or preemption. There is no manual destruction required.

## Internal State & Concurrency
The HeadMotion interface defines no state. State management is the responsibility of the implementing class.

- **State:** Implementations are expected to be stateful, holding data such as the target entity ID, a target world position, or the duration of the motion. This state is almost always mutable.
- **Thread Safety:** Implementations of HeadMotion are **not expected to be thread-safe**. All NPC logic, including the processing of Motion instructions, is designed to be executed exclusively on the main server thread for the world in which the NPC exists. Accessing or modifying a HeadMotion object from any other thread will lead to race conditions and undefined behavior.

## API Surface
HeadMotion inherits its entire API contract from the parent Motion interface. It declares no methods of its own. The API of a typical Motion implementation is as follows.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| update(npc, deltaTime) | MotionResult | O(1) | Executes one tick of the motion logic. Returns a result indicating if the motion is ongoing, completed, or has failed. |
| onStart(npc) | void | O(1) | Called once by the instruction processor before the first call to update. Used for initialization. |
| onEnd(npc) | void | O(1) | Called once when the motion is completed or interrupted. Used for cleanup. |

**Warning:** The exact method signatures are defined in the parent Motion interface and are subject to change. The purpose remains consistent: to manage the state transitions of a discrete NPC action.

## Integration Patterns

### Standard Usage
A developer creates a concrete class implementing HeadMotion to define a new behavior. This class is then instantiated and passed to the NPC's instruction queue to be executed by the server's AI processing loop.

```java
// Example of a concrete implementation being used
// (Note: LookAtEntityMotion is a hypothetical implementation)

LookAtEntityMotion lookAtPlayer = new LookAtEntityMotion(player.getId());
npc.getInstructionQueue().add(lookAtPlayer);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations without Reset:** Creating a reusable, singleton instance of a HeadMotion implementation is dangerous. These objects carry state (like a target) and must be new instances for each use case to avoid behavior contamination between different NPCs or different actions.
- **Blocking Operations:** Never perform blocking operations (e.g., file I/O, database calls) within the update method of a HeadMotion. Doing so will stall the server thread, impacting performance for all entities in that world.

## Data Pipeline
The flow of control and data for a HeadMotion instruction is managed by the server's core NPC update loop.

> Flow:
> AI Behavior Tree -> Selects Action -> Instantiates **LookAtEntityMotion** (implements HeadMotion) -> NPC Instruction Queue -> NPC Update Tick -> **HeadMotion.update()** -> Modifies NPC Entity State (Head Yaw/Pitch) -> State sent to Client Renderers

