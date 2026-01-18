---
description: Architectural reference for InterpretedCombatData
---

# InterpretedCombatData

**Package:** com.hypixel.hytale.server.npc.blackboard.view.combat
**Type:** Transient / Data Transfer Object

## Definition
```java
// Signature
public class InterpretedCombatData {
```

## Architecture & Concepts
InterpretedCombatData is a non-persistent Data Transfer Object (DTO) that represents a simplified, high-level snapshot of an NPC's combat state for a single game tick. It serves as a crucial data structure within the server-side AI system, specifically as part of the Blackboard pattern.

This class acts as a "view" or "interpretation" of the raw, complex data stored in an NPC's AI Blackboard. Instead of various game systems needing to query and understand the low-level details of the AI's decision-making process, a central interpreter populates this object. Systems like animation, sound, and particle effects can then consume this clean, structured data to react to the NPC's current actions.

Its primary architectural role is to **decouple** AI state from state-consuming systems. This prevents systems like the animation controller from needing intimate knowledge of the AI's internal behavior tree or state machine.

## Lifecycle & Ownership
- **Creation:** Instances are created and populated on-demand by a higher-level AI system, likely an interpreter that processes the NPC's Blackboard each tick. The presence of a public `clone` method indicates that copies are frequently made to ensure state immutability for consumers within a single frame.
- **Scope:** Extremely short-lived. An instance of InterpretedCombatData is intended to be valid only for the duration of the game tick in which it was created.
- **Destruction:** The object is eligible for garbage collection as soon as the game tick completes and all systems have finished consuming its data. No long-term references should ever be held.

## Internal State & Concurrency
- **State:** Highly **Mutable**. The object is a plain container of public fields with corresponding setters, designed to be rapidly populated by a single authority (the AI interpreter).
- **Thread Safety:** This class is **not thread-safe**. It contains no synchronization mechanisms and is designed for use exclusively within the main server game loop thread. Accessing or modifying an instance from multiple threads without external locking will result in undefined behavior and data corruption.

## API Surface
The primary API consists of simple getters and setters for its internal fields. The most architecturally significant method is `clone`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| clone() | InterpretedCombatData | O(1) | Creates a new instance with a shallow copy of all field values. Critical for providing a stable, point-in-time snapshot of combat state to multiple consumer systems. |

## Integration Patterns

### Standard Usage
This object is not retrieved from a service locator or context. It is typically created and passed directly to systems that need to react to an NPC's actions during the current tick.

```java
// Inside an AI or NPC update loop
void processNpcCombatState(Blackboard blackboard) {
    // 1. An interpreter creates and populates the data object
    InterpretedCombatData combatData = AIStateInterpreter.interpret(blackboard);

    // 2. The object is passed to other systems for consumption
    animationSystem.update(npc, combatData);
    soundSystem.update(npc, combatData);
    // ...etc
}
```

### Anti-Patterns (Do NOT do this)
- **State Caching:** Do not store a reference to an InterpretedCombatData object across multiple ticks. The data represents a snapshot and will become stale immediately. Always request or receive a new instance on each update cycle.
- **External Modification:** Only the authoritative AI interpretation system should modify the state of this object. Other systems should treat it as read-only to prevent state corruption.
- **Cross-Thread Access:** Never share an instance of this class across threads without implementing external, robust locking. It is fundamentally designed for single-threaded access.

## Data Pipeline
InterpretedCombatData functions as a temporary, structured message that translates raw AI state into actionable information for other engine systems.

> Flow:
> NPC AI Blackboard (Raw State) -> AI State Interpreter -> **InterpretedCombatData** -> Animation System / Sound System / Effects System

