---
description: Architectural reference for WorldTransactionRecord
---

# WorldTransactionRecord

**Package:** com.hypixel.hytale.builtin.adventure.objectives.transaction
**Type:** Specialized Implementation

## Definition
```java
// Signature
public class WorldTransactionRecord extends TransactionRecord {
```

## Architecture & Concepts
The WorldTransactionRecord is a specialized, non-operational implementation of the TransactionRecord base class. It serves as a marker or placeholder within the Adventure Objective system for world-state changes that are either irreversible or managed outside the standard transactional lifecycle.

Unlike other TransactionRecord types that encapsulate reversible actions (e.g., giving a player an item), this class explicitly represents an event that cannot be reverted, completed, or serialized through the objective system's transaction manager. Its primary purpose is to allow the objective system to acknowledge and log a world-level event as part of a quest's transaction history without imposing the overhead or guarantees of a true transactional operation.

Examples of such events include server-wide announcements, weather changes, or the triggering of a complex world event whose state is managed by a separate, non-transactional system. By providing empty implementations for core lifecycle methods, it signals to the transaction manager that no action is required for rollback or completion.

## Lifecycle & Ownership
- **Creation:** Instantiated by high-level Adventure Objective logic when a non-transactional world event occurs that must be tracked as part of a quest's progress. It is never created in response to direct player inventory or state changes.
- **Scope:** The object's lifetime is strictly bound to the parent transaction it is added to. It is a short-lived, transient object.
- **Destruction:** The instance is eligible for garbage collection as soon as the parent transaction is finalized (committed or discarded). The explicit override of shouldBeSerialized to return false ensures it is never persisted to disk, reinforcing its transient nature.

## Internal State & Concurrency
- **State:** The WorldTransactionRecord is stateless. It inherits fields from its parent but adds no new state of its own. Its behavior is defined entirely by its static method overrides, making it effectively an immutable marker object.
- **Thread Safety:** This class is inherently **thread-safe**. As a stateless and immutable object, it can be safely passed between threads. However, the TransactionManager that consumes it is likely not thread-safe and must be accessed from the main game thread.

## API Surface
The public API consists entirely of overrides from the TransactionRecord base class. The intent of these overrides is to nullify transactional behavior.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| revert() | void | O(1) | No-op. This transaction type is irreversible. Calling this method has no effect. |
| complete() | void | O(1) | No-op. This transaction type has no finalization logic. |
| unload() | void | O(1) | No-op. This record holds no resources that require explicit unloading. |
| shouldBeSerialized() | boolean | O(1) | Always returns **false**. Prevents the transaction manager from attempting to persist this record. |

## Integration Patterns

### Standard Usage
This class is not intended for direct instantiation by most developers. It is used by the core objective system to represent irreversible world events within a transactional context.

```java
// Within an objective's state machine logic
TransactionManager transactionManager = objective.getTransactionManager();

// A world event occurs that cannot be undone (e.g., a cinematic trigger)
TransactionRecord worldEventMarker = new WorldTransactionRecord();

// The event is added to the history, but no revert/complete logic is associated
transactionManager.addRecord(worldEventMarker);
```

### Anti-Patterns (Do NOT do this)
- **Expecting Reversibility:** Do not use this class for actions that must be reversible. Its entire purpose is to signal that an action is non-transactional. For reversible actions, create a custom subclass of TransactionRecord with proper revert logic.
- **Adding State or Logic:** Do not subclass WorldTransactionRecord to add state or behavior. If you need custom logic, extend the base TransactionRecord directly. This class is a sealed concept.
- **Forcing Serialization:** Do not override shouldBeSerialized to return true. The game engine is not designed to handle the persistence of this record type and doing so will lead to undefined behavior or data corruption on world load.

## Data Pipeline
The WorldTransactionRecord acts as a terminal data object within a larger flow. It does not process data but rather represents a finalized event.

> Flow:
> External Game Event (e.g., World State Change) -> Adventure Objective Logic -> **WorldTransactionRecord Instantiation** -> Transaction Manager -> Objective State Finalization (Record is discarded)

