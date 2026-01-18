---
description: Architectural reference for SensorHasInteracted
---

# SensorHasInteracted

**Package:** com.hypixel.hytale.server.npc.corecomponents.interaction
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorHasInteracted extends SensorBase {
```

## Architecture & Concepts
SensorHasInteracted is a specialized conditional component within the server-side NPC AI framework. Its primary function is to act as a one-shot trigger that detects if an NPC has recently been interacted with by a specific target entity.

Unlike sensors that continuously monitor a state (e.g., IsTargetInRange), this sensor is event-driven. It checks for a flag that is set by the core interaction system and, upon a successful check, consumes the flag to prevent the condition from remaining true on subsequent AI ticks. This "consume-on-read" pattern ensures that behaviors linked to this sensor fire exactly once per interaction event.

It is a fundamental building block for creating reactive NPC behaviors, such as an NPC responding verbally after a player clicks on them, or a quest-giver acknowledging the player's interaction before presenting a dialog. The sensor also includes a critical safety check to ensure the interacting entity has not died between the interaction event and the sensor evaluation.

## Lifecycle & Ownership
- **Creation:** Instances of SensorHasInteracted are not created directly. They are instantiated by the NPC AI system, typically via a BuilderSensorBase during the assembly of an NPC's behavior profile from archetype definitions.
- **Scope:** The lifecycle of a SensorHasInteracted instance is tightly coupled to the NPC entity it belongs to. It persists as long as the NPC is active in the world.
- **Destruction:** The object is marked for garbage collection when the parent NPC entity is unloaded or destroyed. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** This class is stateless. It contains no mutable fields and all necessary context is passed into the *matches* method during the AI update cycle. The state it evaluates and modifies is managed externally by the Role object, specifically within its StateSupport component.
- **Thread Safety:** This component is **not thread-safe** and must only be accessed from the main server thread responsible for ticking the NPC's world region. The *matches* method performs a read-modify-write operation on the NPC's interaction state via *consumeInteraction*, which would cause race conditions if called concurrently.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Evaluates if the NPC's interaction target has a pending, unconsumed interaction event. Returns false if no target exists, the target is dead, or the interaction has already been consumed. |
| getSensorInfo() | InfoProvider | O(1) | Returns null. This sensor does not provide extended debugging information. |

## Integration Patterns

### Standard Usage
This sensor is not intended to be invoked directly by game logic. It is configured within an NPC's definition and is evaluated automatically by the NPC's AI behavior processor. The following is a conceptual representation of its place in a behavior tree.

```java
// Conceptual example within an NPC Behavior Tree definition
BehaviorTreeBuilder builder = new BehaviorTreeBuilder();

builder.selector()
    .sequence()
        // The AI system evaluates this sensor each tick
        .condition(new SensorHasInteracted(...)) 
        // If the sensor returns true, this action runs
        .action(new PlayAnimation("Wave"))
        .action(new Say("Hello there!"))
    .end()
    // ... other behaviors
.end();
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new SensorHasInteracted()`. The object must be configured and instantiated through its corresponding builder to ensure properties from SensorBase are correctly initialized.
- **Stateful Reliance:** Do not design logic that expects this sensor to return true for multiple consecutive ticks after a single interaction. The `consumeInteraction` call explicitly makes it a single-fire trigger.
- **External Invocation:** Never call the *matches* method from outside the NPC's dedicated AI update loop. It relies on the specific context and state provided during that phase.

## Data Pipeline
The flow of data for an interaction event is managed across several systems, with this sensor acting as the final gatekeeper for the AI.

> Flow:
> Player Interaction Packet -> Server Interaction System -> **Role.StateSupport** (Interaction Flag Set) -> NPC AI Tick -> **SensorHasInteracted.matches()** (Reads & Consumes Flag) -> Behavior Tree -> NPC Action (e.g., Speak)

