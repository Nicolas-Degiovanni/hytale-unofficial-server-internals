---
description: Architectural reference for FlockPlayerMembership
---

# FlockPlayerMembership

**Package:** com.hypixel.hytale.server.npc.movement
**Type:** Type-Safe Enum

## Definition
```java
// Signature
public enum FlockPlayerMembership implements Supplier<String> {
```

## Architecture & Concepts
FlockPlayerMembership is a foundational enumeration within the server-side NPC movement and artificial intelligence subsystem. Its primary architectural role is to provide a compile-time, type-safe mechanism for defining and querying a player's relationship to an NPC flock.

This enum acts as a **state specifier** or **filter criteria** for higher-level AI behaviors. By replacing ambiguous booleans or "magic strings" (e.g., "is_member"), it enforces a strict contract for any system that needs to make decisions based on player flocking status. This significantly improves code clarity and eliminates a class of potential runtime errors in complex AI logic, such as target selection, threat assessment, or coordinated group movements.

It is a leaf component in the AI system, providing immutable state definitions rather than executing logic itself.

## Lifecycle & Ownership
- **Creation:** Enum constants are instantiated automatically by the Java Virtual Machine (JVM) during the class loading phase. This process is guaranteed to happen only once, before any code can access the enum constants.
- **Scope:** Application-wide. The instances Member, NotMember, and Any are static, final, and persist for the entire lifetime of the server process.
- **Destruction:** The enum and its constants are unloaded from memory only when the JVM shuts down. There is no manual destruction or garbage collection of these instances during runtime.

## Internal State & Concurrency
- **State:** **Immutable**. Each enum constant holds a final *description* string, which is initialized at creation and can never be modified. The state of the enum itself is fixed at compile time.
- **Thread Safety:** **Inherently thread-safe**. Due to its immutable nature and the JVM's guarantees for enum initialization, FlockPlayerMembership can be safely accessed and referenced from any thread without requiring locks or other synchronization primitives.

## API Surface
The primary API consists of the enum constants themselves, which are used for assignment and comparison.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | String | O(1) | Returns the human-readable description of the enum constant. Implements the Supplier interface. |

## Integration Patterns

### Standard Usage
This enum should be used for direct, type-safe comparisons within AI logic to determine behavior paths.

```java
// How a developer should normally use this
public void updateBehavior(Player target, Flock flock) {
    FlockPlayerMembership status = flock.getPlayerMembership(target);

    if (status == FlockPlayerMembership.Member) {
        // Initiate supportive or coordinated behavior
        executeFollowRoutine(target);
    } else {
        // Initiate defensive or aggressive behavior
        executeThreatAssessment(target);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **String-based Comparison:** Never use the string value for logical comparisons. This defeats the purpose of a type-safe enum and re-introduces the risk of errors from string manipulation or typos.
  ```java
  // INCORRECT - Brittle and inefficient
  if (status.get().equals("Player is member of a flock")) {
      // ...
  }
  ```
- **Null Assignment:** Logic should not be designed to handle a null FlockPlayerMembership. A variable of this type should always hold one of the three defined constants. Defensive null checks indicate a potential design flaw in the calling code.

## Data Pipeline
FlockPlayerMembership does not process data. Instead, it serves as a conditional gate or filter within a larger data processing pipeline, such as an AI behavior tree or state machine.

> Flow:
> AI Tick Event -> Behavior Tree Node -> Query Player Status -> **FlockPlayerMembership** (Filter) -> Select Behavior Branch -> Generate Movement Command

