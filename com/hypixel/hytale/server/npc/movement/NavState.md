---
description: Architectural reference for NavState
---

# NavState

**Package:** com.hypixel.hytale.server.npc.movement
**Type:** Enum

## Definition
```java
// Signature
public enum NavState implements Supplier<String> {
```

## Architecture & Concepts
NavState is a foundational enumeration that defines the possible states of an NPC's navigation or pathfinding process. It functions as a type-safe state machine definition for the server-side movement system.

Architecturally, NavState decouples the pathfinding algorithm from the behavior controllers that consume its results. By providing a fixed set of explicit states—such as PROGRESSING, BLOCKED, or AT_GOAL—it eliminates ambiguity and prevents the use of error-prone "magic values" like integers or strings for state tracking. Its implementation of the Supplier interface provides a standardized way to retrieve a human-readable description, primarily for logging and debugging purposes.

This enum is a critical component for any system that needs to query, react to, or manage the status of an entity's movement operation.

## Lifecycle & Ownership
- **Creation:** All instances of the NavState enum (INIT, PROGRESSING, etc.) are constructed and initialized by the Java Virtual Machine (JVM) during class loading. This process is automatic and occurs before any game code can access the enum.
- **Scope:** As static final constants, NavState instances exist for the entire lifetime of the server application. They are globally accessible and shared across all threads.
- **Destruction:** The instances are reclaimed by the JVM only when the application shuts down and the class loader is garbage collected. Manual destruction is neither possible nor necessary.

## Internal State & Concurrency
- **State:** The internal state of each NavState constant is immutable. The *description* field is a final string assigned during JVM initialization and cannot be changed at runtime.
- **Thread Safety:** NavState is inherently thread-safe. Its immutability and JVM-managed lifecycle guarantee that it can be safely accessed and compared from any thread without synchronization or locking mechanisms.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | String | O(1) | Returns the human-readable description of the state. Intended for logging or debugging. |

## Integration Patterns

### Standard Usage
NavState is used to control logic flow within movement and AI behavior systems by comparing the current state of a navigation agent to one of the predefined constants.

```java
// A hypothetical MovementController checking an agent's status
NavigationAgent agent = npc.getNavigationAgent();

if (agent.getCurrentState() == NavState.AT_GOAL) {
    // Trigger arrival behavior, e.g., interact with a block
    npc.getBehaviorTree().onGoalReached();
} else if (agent.getCurrentState() == NavState.BLOCKED) {
    // Attempt to find an alternative path or abort
    agent.requestRepath();
}
```

### Anti-Patterns (Do NOT do this)
- **Comparison by String:** Never use the string description for logical comparisons. This is brittle, inefficient, and defeats the purpose of a type-safe enum.

```java
// INCORRECT: This will break if the description text is ever changed.
if (agent.getCurrentState().get().equals("Reached target")) {
    // ...
}

// CORRECT: Use direct object comparison.
if (agent.getCurrentState() == NavState.AT_GOAL) {
    // ...
}
```

- **Reliance on Ordinal:** Do not use the `ordinal()` method for serialization or logic. The order of enum constants may change, which would break saved data or conditional logic.

## Data Pipeline
NavState acts as a status flag or output signal from the core pathfinding system. It does not process data itself but represents the result of a data processing operation.

> Flow:
> Pathfinding Algorithm -> **NavState** -> Behavior Tree / Movement Controller -> Entity Action

