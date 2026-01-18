---
description: Architectural reference for Positioning
---

# Positioning

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator
**Type:** Static Enum

## Definition
```java
// Signature
public enum Positioning {
```

## Architecture & Concepts
The Positioning enum provides a type-safe, constrained set of values for describing the relative spatial orientation between two entities in a combat context. It is a fundamental building block of the NPC Combat Action Evaluator system.

This enumeration serves as a data contract, eliminating the use of "magic values" such as integers or strings for position checks. By using Positioning, the combat system can define clear, readable, and compile-time-verified requirements for abilities and behaviors. For example, an ability like "Backstab" would explicitly require the `Behind` constant, while a standard attack might only require `Front` or `Flank`.

It is a passive data structure and does not contain any logic itself; rather, it is produced by spatial analysis systems and consumed by decision-making algorithms.

### Lifecycle & Ownership
-   **Creation:** Instances are created and managed exclusively by the Java Virtual Machine (JVM) during class loading. Application code does not and cannot instantiate these objects.
-   **Scope:** The enum constants exist as singleton instances for the entire lifetime of the application.
-   **Destruction:** Resources are reclaimed by the JVM during application shutdown. There is no manual cleanup required.

## Internal State & Concurrency
-   **State:** Inherently immutable. The defined constants are static, final instances whose state cannot be modified at runtime.
-   **Thread Safety:** Fully thread-safe. As immutable singletons, instances of Positioning can be safely accessed, passed, and compared from any thread without requiring external synchronization or locks.

## API Surface
The primary API consists of the enum constants themselves, which are used for assignment and comparison.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Any | Positioning | N/A | Represents no specific positional requirement. |
| Front | Positioning | N/A | Represents a position within the frontal arc of a target. |
| Behind | Positioning | N/A | Represents a position within the rear arc of a target. |
| Flank | Positioning | N/A | Represents a position to the side of a target, neither front nor behind. |

## Integration Patterns

### Standard Usage
Positioning is typically used in conditional logic within AI behavior trees or action evaluators to determine if an entity meets the spatial requirements to execute a specific combat action.

```java
// Example: Evaluating a 'Backstab' action
public boolean canExecuteBackstab(Entity attacker, Entity target) {
    // A hypothetical system calculates the relative position
    Positioning relativePos = SpatialSystem.getRelativePosition(attacker, target);

    // The enum is used for a direct, type-safe comparison
    return relativePos == Positioning.Behind;
}
```

### Anti-Patterns (Do NOT do this)
-   **Ordinal Reliance:** Do not use the `ordinal()` method for serialization or network replication. The integer order of enum constants is not guaranteed to be stable across different versions of the game client or server. Use the `name()` method for a stable, string-based representation.
-   **Null Usage:** Avoid passing null in place of a Positioning value. Systems consuming this enum are designed to operate on its defined constants and may not perform defensive null checks, leading to a NullPointerException.
-   **Reflective Instantiation:** Do not attempt to create new instances of this enum via reflection. This practice breaks fundamental JVM guarantees and will result in unpredictable and unstable system behavior.

## Data Pipeline
Positioning acts as a discrete data point in the AI decision-making pipeline. It is the output of one stage and the input for the next.

> Flow:
> World State (Entity Positions/Rotations) -> Spatial Analysis System -> **Positioning** (enum value) -> NPC Combat Action Evaluator -> Final Action Selection

