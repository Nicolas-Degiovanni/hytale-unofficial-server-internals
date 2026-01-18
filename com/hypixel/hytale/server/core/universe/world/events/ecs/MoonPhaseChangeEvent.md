---
description: Architectural reference for MoonPhaseChangeEvent
---

# MoonPhaseChangeEvent

**Package:** com.hypixel.hytale.server.core.universe.world.events.ecs
**Type:** Data Transfer Object / Event

## Definition
```java
// Signature
public class MoonPhaseChangeEvent extends EcsEvent {
```

## Architecture & Concepts
The MoonPhaseChangeEvent is an immutable, transient data structure that functions as a notification within the server-side Entity-Component-System (ECS) framework. Its sole purpose is to signal a change in the celestial moon phase to any interested game systems.

As a subclass of EcsEvent, it acts as a message on the central event bus. This design decouples the system responsible for calculating time and celestial mechanics from the systems that must react to those changes. For example, a system managing nocturnal creature behavior does not need to know *how* the moon phase is calculated; it only needs to subscribe to this event to be notified when a change occurs. This promotes a clean, reactive, and highly modular architecture.

The event is designed to be a "fire-and-forget" message, carrying the essential new state—the integer representing the new moon phase—to all listeners simultaneously.

### Lifecycle & Ownership
- **Creation:** Instantiated by a high-level world simulation system, likely a TimeOfDaySystem or a dedicated CelestialSystem, at the exact game tick when the moon's phase transitions.
- **Scope:** Extremely short-lived. The object exists only for the duration of the event dispatch cycle within a single server tick. It is not intended to be stored or referenced beyond the scope of an event handler method.
- **Destruction:** The object becomes eligible for garbage collection immediately after the event bus has finished dispatching it to all subscribers for the current tick. No system should retain a reference to it.

## Internal State & Concurrency
- **State:** Immutable. The internal state, newMoonPhase, is a final field set only once during construction. This guarantees that the event's data cannot be mutated after its creation, which is a critical property for ensuring predictable behavior in an event-driven system.
- **Thread Safety:** Inherently thread-safe due to its immutability. Multiple systems can read from this event object across different threads (if the ECS engine is multi-threaded) without any risk of data corruption or race conditions. The object itself poses no concurrency hazards.

## API Surface
The public contract is minimal, focusing exclusively on data retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getNewMoonPhase() | int | O(1) | Retrieves the integer value representing the new moon phase. |

## Integration Patterns

### Standard Usage
Systems subscribe to this event type through the ECS event bus. When the event is fired, the system's registered handler method is invoked with the event object as an argument.

```java
// Example: A system that makes werewolves stronger during a full moon.
// This method would be registered with the event bus.

public void onMoonPhaseChange(MoonPhaseChangeEvent event) {
    int phase = event.getNewMoonPhase();
    if (phase == FULL_MOON_PHASE) {
        // Apply "Full Moon" component to all werewolf entities
        this.world.query(Werewolf.class).forEach(entity -> {
            entity.addComponent(new FullMoonFrenzyComponent());
        });
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Storage:** Do not store a reference to a MoonPhaseChangeEvent object in a component or system field. The event represents a point-in-time notification, not persistent state. Copy the integer value if it must be stored.
- **Manual Instantiation by Listeners:** Systems that consume this event must never create an instance of it. Doing so would inject a false state change into the game world and is a severe violation of the event flow pattern.

## Data Pipeline
The flow of this event through the server architecture is linear and predictable. It serves to broadcast a state change from a single authoritative source to multiple, decoupled consumers.

> Flow:
> World Time System -> Calculates new phase -> **new MoonPhaseChangeEvent()** -> ECS Event Bus -> (Broadcast) -> CreatureBehaviorSystem, LightingSystem, OceanTideSystem

