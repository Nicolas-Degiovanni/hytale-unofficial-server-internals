---
description: Architectural reference for ReservationStatus
---

# ReservationStatus

**Package:** com.hypixel.hytale.server.npc.blackboard.view.interaction
**Type:** Utility

## Definition
```java
// Signature
public enum ReservationStatus {
   NOT_RESERVED,
   RESERVED_OTHER,
   RESERVED_THIS;
}
```

## Architecture & Concepts
ReservationStatus is a type-safe enumeration that represents the exclusive access state of an interactable entity or point within the world, as perceived by an NPC's AI. It is a core component of the NPC Blackboard system, specifically for views that manage interaction logic.

This enum provides a simple, immutable, and unambiguous contract for determining whether a resource (like a workstation, a conversation partner, or a tactical position) is available. It is fundamental to preventing multiple AI agents from attempting to use the same resource simultaneously, which would lead to conflicting or broken behaviors. Its primary role is to serve as a state flag within a behavior tree or state machine, allowing an agent to make decisions based on resource availability.

## Lifecycle & Ownership
- **Creation:** The Java Virtual Machine (JVM) creates instances for each enum constant (NOT_RESERVED, RESERVED_OTHER, RESERVED_THIS) during class loading. These instances are compile-time constants.
- **Scope:** These instances are static and persist for the entire lifetime of the server application. They are effectively global singletons.
- **Destruction:** The instances are reclaimed only when the application's class loader is garbage collected, which typically occurs at server shutdown.

## Internal State & Concurrency
- **State:** Enum instances are deeply immutable. Their state is defined at compile time and cannot be altered at runtime.
- **Thread Safety:** ReservationStatus is inherently thread-safe. As immutable singletons, its values can be safely read and compared from any thread without synchronization. This is a critical guarantee, as multiple NPC behavior trees may execute concurrently and query the same blackboard state.

## API Surface
The public contract of an enum consists of its constant values.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| NOT_RESERVED | ReservationStatus | O(1) | Indicates the target resource is available for reservation. |
| RESERVED_OTHER | ReservationStatus | O(1) | Indicates the target is reserved by a different agent. |
| RESERVED_THIS | ReservationStatus | O(1) | Indicates the target is reserved by the querying agent. |

## Integration Patterns

### Standard Usage
This enum is intended to be used in conditional logic within AI behavior trees or decision-making systems to control access to shared resources.

```java
// How a developer should normally use this
BlackboardView view = agent.getBlackboard().getView(InteractionView.class);
ReservationStatus status = view.getReservationStatus(targetEntity);

if (status == ReservationStatus.NOT_RESERVED) {
    // Attempt to reserve and use the target
    view.reserve(targetEntity);
} else if (status == ReservationStatus.RESERVED_THIS) {
    // Continue with the interaction
    agent.performInteraction();
}
```

### Anti-Patterns (Do NOT do this)
- **Ordinal Comparison:** Do not use the ordinal() method for logic or serialization (e.g., `if (status.ordinal() == 0)`). This is extremely brittle and will break if the enum order is ever changed. Always compare instances directly.
- **Null Checks:** A method returning a ReservationStatus should never return null. The NOT_RESERVED constant should be used to represent the absence of a reservation. Code should not be written to defensively handle a null ReservationStatus.

## Data Pipeline
ReservationStatus is not a data processor but a state value used within a larger data flow. It represents the outcome of a state query.

> Flow:
> AI Behavior Tree Node -> Blackboard Query -> **ReservationStatus** -> Decision Logic -> NPC Action Command

