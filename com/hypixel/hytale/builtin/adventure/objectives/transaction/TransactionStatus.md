---
description: Architectural reference for TransactionStatus
---

# TransactionStatus

**Package:** com.hypixel.hytale.builtin.adventure.objectives.transaction
**Type:** Utility

## Definition
```java
// Signature
enum TransactionStatus {
```

## Architecture & Concepts
TransactionStatus is a type-safe enumeration that represents the terminal state of an objective-related transaction. It serves as a fundamental building block within the adventure mode's state machine, providing a clear, self-documenting, and robust mechanism for signaling the outcome of an operation.

Its primary architectural role is to eliminate "magic values" (e.g., integers like 0 for success, -1 for fail) or string comparisons. By enforcing a constrained set of possible outcomes at compile time, it reduces runtime errors and improves the reliability of systems that consume transaction results, such as the Quest and Objective tracking services.

## Lifecycle & Ownership
- **Creation:** As an enumeration, TransactionStatus instances are created by the Java Virtual Machine during class loading. The constants SUCCESS and FAIL are instantiated once and exist as singletons for the entire application lifecycle.
- **Scope:** The scope is global and static. These constants are available as soon as the TransactionStatus enum is loaded by the classloader.
- **Destruction:** The enum and its constants are unloaded only when the JVM shuts down. There is no manual memory management or destruction required.

## Internal State & Concurrency
- **State:** Inherently immutable. The state of an enum constant cannot be changed after its creation.
- **Thread Safety:** TransactionStatus is unconditionally thread-safe. Its constants can be safely accessed and passed between any number of threads without synchronization. This is a guaranteed property of the Java language specification for enumerations.

## API Surface
The API consists solely of the predefined constants.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SUCCESS | TransactionStatus | O(1) | Represents a transaction that was completed successfully. |
| FAIL | TransactionStatus | O(1) | Represents a transaction that failed due to validation errors, insufficient resources, or other conditions. |

## Integration Patterns

### Standard Usage
TransactionStatus is typically used as a return type or as a field within a more complex result object to communicate the outcome of a transactional operation.

```java
// A service method returns the status directly
public TransactionStatus completeObjective(Objective objective) {
    if (player.hasRequiredItem(objective.getItem())) {
        // ... logic to complete objective
        return TransactionStatus.SUCCESS;
    }
    return TransactionStatus.FAIL;
}
```

### Anti-Patterns (Do NOT do this)
- **Ordinal Comparison:** Do not rely on the ordinal value of the enum for logic. This is brittle and can break if the enum order is ever changed. Always compare instances directly.

```java
// BAD: Prone to breaking if enum order changes
if (status.ordinal() == 0) { /* ... */ }

// GOOD: Robust and readable
if (status == TransactionStatus.SUCCESS) { /* ... */ }
```

## Data Pipeline
TransactionStatus does not process data itself; it is the data. It represents the final state produced by a processing component.

> Flow:
> ObjectiveService.attemptComplete() -> Transaction Logic -> **TransactionStatus** -> Quest System State Update

