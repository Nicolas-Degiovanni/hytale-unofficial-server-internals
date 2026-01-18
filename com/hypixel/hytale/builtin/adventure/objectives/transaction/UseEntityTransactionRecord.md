---
description: Architectural reference for UseEntityTransactionRecord
---

# UseEntityTransactionRecord

**Package:** com.hypixel.hytale.builtin.adventure.objectives.transaction
**Type:** Transient Command Object

## Definition
```java
// Signature
public class UseEntityTransactionRecord extends TransactionRecord {
```

## Architecture & Concepts
The UseEntityTransactionRecord is a concrete implementation of the *Command* design pattern, operating within the Adventure Objective transactional framework. It represents a single, ephemeral operation: the temporary reservation of a world entity for a specific objective task.

Its primary architectural role is to act as a **compensating action**. When a player interacts with a quest-related entity, the objective system may place a temporary lock or claim on that entity to prevent concurrent interactions. This class encapsulates the logic required to *release* that claim.

Critically, the `revert`, `complete`, and `unload` methods all perform the identical action of removing the entity task from the ObjectiveDataStore. This design ensures that the entity reservation is released regardless of the parent transaction's outcome, preventing orphaned entity locks. The non-serializable nature of this record, enforced by `shouldBeSerialized` returning false, designates it as a purely in-memory, session-bound state manager.

### Lifecycle & Ownership
- **Creation:** Instantiated internally by the Adventure Objective engine when a player's action necessitates the temporary reservation of an entity for a quest task. It is immediately added to a parent Transaction.
- **Scope:** Extremely short-lived. Its lifecycle is strictly bound to its parent Transaction object. It exists only for the duration of the in-memory transaction.
- **Destruction:** The object is dereferenced and becomes eligible for garbage collection as soon as its parent Transaction is resolved (either through `revert`, `complete`, or `unload`). There is no manual destruction required.

## Internal State & Concurrency
- **State:** The internal state, consisting of `objectiveUUID` and `taskId`, is set at construction and is effectively immutable for the lifetime of the object. This class is a simple data carrier for its compensating logic.
- **Thread Safety:** This class is **not thread-safe** and contains no internal synchronization mechanisms. It is designed to be created, owned, and executed exclusively by the single-threaded Adventure Objective system. Concurrent access from multiple threads is a severe design violation and will lead to race conditions within the ObjectiveDataStore.

## API Surface
The public API is a strict implementation of the TransactionRecord contract, intended to be invoked only by a parent Transaction controller.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| revert() | void | O(1) | Releases the entity task reservation from the central data store. |
| complete() | void | O(1) | Releases the entity task reservation from the central data store. |
| unload() | void | O(1) | Releases the entity task reservation from the central data store. |
| shouldBeSerialized() | boolean | O(1) | Always returns false, preventing persistence of this transient record. |

## Integration Patterns

### Standard Usage
A developer will never interact with this class directly. It is an internal component of the objective system. The following demonstrates its intended use by the engine.

```java
// PSEUDOCODE: Within the objective engine's logic

// A parent transaction is initiated for a player action
Transaction parentTransaction = new Transaction();
UUID objectiveId = getCurrentObjectiveId();
String entityTaskId = getTargetEntityTaskId();

// The system reserves the entity and creates this record to ensure its eventual release
ObjectivePlugin.get().getObjectiveDataStore().addEntityTask(objectiveId, entityTaskId);
parentTransaction.addRecord(new UseEntityTransactionRecord(objectiveId, entityTaskId));

// ... other operations are added to the transaction ...

// When the overall operation is finalized, the transaction is resolved,
// which automatically invokes the appropriate method on this record.
if (operationFailed) {
    parentTransaction.revert(); // Triggers UseEntityTransactionRecord.revert()
} else {
    parentTransaction.complete(); // Triggers UseEntityTransactionRecord.complete()
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of UseEntityTransactionRecord in game or plugin logic. Its management is the sole responsibility of the core objective transaction engine.
- **Manual Invocation:** Do not call `revert`, `complete`, or `unload` directly. These are lifecycle callbacks managed by a parent Transaction object. Calling them manually will desynchronize the state of the transaction and the ObjectiveDataStore, likely causing quest-breaking bugs.
- **Persistence:** Do not attempt to serialize or otherwise persist this object. It is explicitly designed to be transient. Attempting to save it will fail and, if forced, would lead to invalid state and dangling entity references upon loading.

## Data Pipeline
This component does not process a flow of data. Instead, it represents a step in a *control flow* designed to guarantee state consistency.

> Flow:
> Player Interaction with Quest Entity -> Objective Engine Logic -> **UseEntityTransactionRecord (Creation & Enqueue)** -> Parent Transaction Resolution -> **UseEntityTransactionRecord (Execution)** -> ObjectiveDataStore (Entity lock released)

