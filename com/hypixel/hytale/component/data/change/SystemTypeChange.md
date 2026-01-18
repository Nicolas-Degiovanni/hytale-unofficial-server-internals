---
description: Architectural reference for SystemTypeChange
---

# SystemTypeChange

**Package:** com.hypixel.hytale.component.data.change
**Type:** Transient Data Object

## Definition
```java
// Signature
public class SystemTypeChange<ECS_TYPE, T extends ISystem<ECS_TYPE>> implements DataChange {
```

## Architecture & Concepts
The SystemTypeChange class is an immutable data structure that represents a command to alter the state of the core Entity-Component-System (ECS) engine. It encapsulates a single, atomic change: the addition or removal of a specific ISystem from the game's processing loop.

This class is a critical component of the engine's dynamic world behavior. Instead of systems being statically defined at compile time, SystemTypeChange objects allow for runtime modification of game logic. For example, entering a new zone or starting a minigame can trigger the creation of these objects to activate or deactivate relevant systems like a QuestTrackingSystem or a ScoreboardSystem.

Architecturally, it follows the **Command Pattern**. It decouples the invoker of a change (e.g., a GameModeManager) from the receiver that executes the change (e.g., the central SystemManager). This promotes a clean, event-driven design where engine subsystems communicate through well-defined, immutable messages rather than direct, stateful method calls.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by high-level game logic that needs to modify the set of active systems. This is typically done in response to a significant game state transition.
- **Scope:** Extremely short-lived. A SystemTypeChange object is designed to be transient and exists only for the duration of its dispatch and processing, usually within a single engine tick. It is a "fire-and-forget" message.
- **Destruction:** The object has no explicit destruction logic. It is garbage collected as soon as it has been consumed by the target system and falls out of scope. Holding long-term references to these objects is an anti-pattern.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields (type, systemType) are declared as final and are initialized exclusively through the constructor. This guarantees that a change command cannot be altered after its creation, preventing a wide class of bugs related to state corruption.
- **Thread Safety:** **Inherently thread-safe**. Due to its immutability, a SystemTypeChange instance can be safely passed between threads without any external locking or synchronization. This is vital for engine architecture where changes might be queued from a network or scripting thread to be processed on the main game thread.

## API Surface
The public contract is minimal, providing read-only access to the encapsulated data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getType() | ChangeType | O(1) | Returns the type of change (e.g., ADD, REMOVE). |
| getSystemType() | SystemType | O(1) | Returns the specific system type this change applies to. |

## Integration Patterns

### Standard Usage
The correct pattern is to create a new instance and immediately dispatch it to a central processing queue or event bus. The creator should not retain a reference to the object.

```java
// Example: Activating a physics system when a world loads
SystemType physicsSystem = Systems.PHYSICS;
SystemTypeChange addPhysics = new SystemTypeChange(ChangeType.ADD, physicsSystem);

// Dispatch the change to the central system manager or event bus
// The dispatcher is responsible for processing this command.
systemChangeDispatcher.dispatch(addPhysics);
```

### Anti-Patterns (Do NOT do this)
- **State Caching:** Do not store instances of SystemTypeChange in member variables or caches. They represent a point-in-time event, not persistent state.
- **Object Re-use:** These objects are lightweight and should not be pooled or re-used. Always create a new instance for each distinct change command to ensure clarity and prevent bugs.
- **Delayed Processing:** While these objects can be queued, they should be processed promptly. A long queue of unprocessed system changes can lead to unpredictable game state and behavior.

## Data Pipeline
The SystemTypeChange object is a data payload that flows from a high-level logic controller to the core ECS engine.

> Flow:
> Game State Manager -> `new SystemTypeChange()` -> System Change Queue -> **SystemManager** -> ECS World State Update

