---
description: Architectural reference for ReservationProvider
---

# ReservationProvider

**Package:** com.hypixel.hytale.server.npc.blackboard.view.interaction
**Type:** Contract Interface

## Definition
```java
// Signature
public interface ReservationProvider {
```

## Architecture & Concepts
The ReservationProvider interface defines a contract for querying the interaction availability of an entity within the server's AI system. It serves as a critical decision-making component, primarily used by Non-Player Character (NPC) behaviors to prevent conflicting actions, such as multiple NPCs attempting to interact with the same player, object, or location simultaneously.

This interface is a key element of the NPC "Blackboard" system, which acts as a shared memory space for AI agents. By abstracting the logic for checking interaction "reservations", the system allows for different reservation strategies to be implemented and swapped without altering the core AI behavior trees or state machines. Implementations of this interface act as gatekeepers, answering the fundamental question: "Is this target entity available for me to interact with right now?"

The method signature, which operates on EntityStore references and a ComponentAccessor, firmly places this interface within the server's Entity Component System (ECS) architecture. It is designed to be a pure, functional query that reads state from the ECS without causing side effects.

## Lifecycle & Ownership
As an interface, ReservationProvider itself has no lifecycle. The following pertains to its concrete implementations.

- **Creation:** Implementations are typically instantiated as stateless, reusable services during the server's bootstrap phase or the initialization of the AI subsystem. They are often registered with a central service locator or dependency injection container.
- **Scope:** Implementations are designed to be long-lived, often persisting for the entire lifetime of the server world. They do not hold per-NPC state and are therefore shared across all AI agents.
- **Destruction:** These objects are cleaned up when the server world is shut down or the AI system is reloaded.

## Internal State & Concurrency
- **State:** The interface is inherently stateless. Implementations are **strongly expected** to be stateless. All required context for a decision—the querier, the target, and the means to access their components—is passed directly into the getReservationStatus method. This design avoids state management complexity within the provider itself.
- **Thread Safety:** Implementations must be thread-safe. Given their stateless, functional design, they are generally safe for concurrent access by default, provided the underlying ECS operations (via ComponentAccessor) are themselves thread-safe. The engine's AI update loop may run across multiple threads, making this a non-negotiable requirement.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getReservationStatus(querier, target, accessor) | ReservationStatus | O(1) | Queries the ECS to determine if the target entity is reserved. Returns an enum indicating availability. This is a read-only operation. |

## Integration Patterns

### Standard Usage
This interface is not intended to be used directly by general game logic. Its primary consumer is the AI Behavior Tree system. A node within a behavior tree will query a ReservationProvider implementation before executing an interaction task.

```java
// Within an AI Behavior Node's update logic
ReservationProvider provider = aiContext.getService(ReservationProvider.class);
Ref<EntityStore> self = getOwnEntityRef();
Ref<EntityStore> target = getInteractionTargetRef();

// Check if the target is available before proceeding
ReservationStatus status = provider.getReservationStatus(self, target, world.getComponentAccessor());

if (status == ReservationStatus.AVAILABLE) {
    // Proceed with interaction logic...
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Creating an implementation that stores internal state related to a specific NPC or query is a severe anti-pattern. This breaks reusability and introduces thread-safety issues.
- **Causing Side Effects:** The getReservationStatus method must not modify any components or world state. It is a query, not a command. The action of *making* a reservation is handled by a separate system, likely an action or task that is executed after a successful availability check.

## Data Pipeline
The ReservationProvider acts as a conditional gate in an AI's decision-making flow rather than a step in a data transformation pipeline.

> Flow:
> AI Behavior Tree Tick -> Interaction Node Evaluation -> **ReservationProvider.getReservationStatus()** -> Decision (Success/Failure) -> AI Action Execution or Abort

