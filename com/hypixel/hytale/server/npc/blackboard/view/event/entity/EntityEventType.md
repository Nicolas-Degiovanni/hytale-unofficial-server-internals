---
description: Architectural reference for EntityEventType
---

# EntityEventType

**Package:** com.hypixel.hytale.server.npc.blackboard.view.event.entity
**Type:** Enumeration / Type-Safe Constant

## Definition
```java
// Signature
public enum EntityEventType implements Supplier<String> {
```

## Architecture & Concepts
EntityEventType provides a fixed, compile-time-safe set of identifiers for fundamental events that can occur to a server-side entity. Its primary role is within the NPC AI and behavior system, specifically relating to the "Blackboard" pattern, where it acts as a key or trigger for behavior tree evaluation.

By defining a constrained set of event types, this enum prevents the use of raw strings for event handling, a practice that is highly susceptible to runtime errors from typos and lacks compile-time validation. This class enforces a strict contract for any system that produces or consumes entity-related events.

The implementation of the Java `Supplier<String>` interface is a deliberate design choice to facilitate easy integration with debugging tools, logging frameworks, or administrative user interfaces that require a human-readable representation of the event, without forcing consumers to call a specific getter method.

## Lifecycle & Ownership
- **Creation:** Instances of this enum (DAMAGE, DEATH, INTERACTION) are constructed and initialized once by the Java Virtual Machine during class loading. There is no dynamic instantiation.
- **Scope:** The enum constants are static and persist for the entire lifetime of the server application. They are globally accessible and effectively singletons.
- **Destruction:** The instances are garbage collected by the JVM only when the application is shutting down and the class loader is unloaded. Manual destruction is not possible or necessary.

## Internal State & Concurrency
- **State:** Deeply immutable. Each enum constant holds a private, final `description` string that is assigned at creation and can never be modified.
- **Thread Safety:** This class is inherently thread-safe. As immutable, globally unique constants, its instances can be safely passed between and read by multiple threads without any form of synchronization or locking.

## API Surface
The primary API consists of the enum constants themselves, which are used for identity comparison and in control flow statements.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DAMAGE | EntityEventType | O(1) | Represents an event triggered when an entity receives damage. |
| DEATH | EntityEventType | O(1) | Represents an event triggered when an entity's health is reduced to zero or below. |
| INTERACTION | EntityEventType | O(1) | Represents an event triggered by a player or another entity performing a "use" action on this entity. |
| get() | String | O(1) | Returns the human-readable description of the event. Fulfills the Supplier contract. |
| VALUES | EntityEventType[] | O(1) | A cached, static array of all enum constants. **Warning:** Prefer using this over the `values()` method in performance-critical loops to avoid repeated array allocation. |

## Integration Patterns

### Standard Usage
EntityEventType is intended to be used in control flow structures like switch statements within AI or game logic systems to react to specific occurrences.

```java
// Example from an NPC behavior controller
void onEntityEvent(EntityEventType eventType, EventContext context) {
    switch (eventType) {
        case DAMAGE:
            this.npc.getBehaviorTree().trigger("ReactToPain");
            break;
        case INTERACTION:
            this.npc.getDialogueSystem().startConversation(context.getInitiator());
            break;
        case DEATH:
            this.npc.getLootSystem().dropLoot();
            break;
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Comparison via name():** Never compare enum instances using their string names, such as `eventType.name().equals("DAMAGE")`. This is inefficient and defeats the purpose of type safety. Always use direct object comparison: `eventType == EntityEventType.DAMAGE`.
- **Reliance on ordinal():** Do not persist or base logic on the integer value from `ordinal()`. The value is fragile and will change if the declaration order of the enum constants is modified, leading to subtle and severe bugs.
- **Extensibility:** Do not attempt to extend this enum. It is a final, closed set of constants by design. If new event types are needed, they must be added directly to the source file.

## Data Pipeline
This enum does not process data itself; rather, it is a data token that flows through various server systems to signal a specific event.

> Flow:
> Low-Level Engine Event (e.g., HealthComponent update) -> Game Logic System -> Event Dispatcher -> **EntityEventType** as payload -> NPC Blackboard System -> Behavior Tree Evaluation -> NPC Action (e.g., state change)

