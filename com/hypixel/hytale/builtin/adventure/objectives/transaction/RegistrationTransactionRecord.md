---
description: Architectural reference for RegistrationTransactionRecord
---

# RegistrationTransactionRecord

**Package:** com.hypixel.hytale.builtin.adventure.objectives.transaction
**Type:** Transient

## Definition
```java
// Signature
public class RegistrationTransactionRecord extends TransactionRecord {
```

## Architecture & Concepts
The RegistrationTransactionRecord is a specialized implementation of the TransactionRecord pattern. Its sole purpose is to encapsulate an event listener registration action within a transactional boundary, ensuring that temporary listeners can be reliably torn down.

This class acts as a command object for an *un-registration* operation. It holds a functional reference, a BooleanConsumer, which when invoked with *false*, removes a previously registered event listener from the game's EventRegistry. This mechanism is critical for the Adventure Mode objective system, where quest steps may need to register temporary listeners (e.g., "listen for player entering area X"). These listeners must be cleaned up regardless of whether the objective is completed, failed, or unloaded, preventing memory leaks and unintended gameplay behavior.

A key design decision is that this record is non-serializable, as indicated by the `shouldBeSerialized` method returning false. This firmly establishes it as a runtime-only, in-memory construct for managing transient state, not for persisting game progress.

## Lifecycle & Ownership
-   **Creation:** Instances are not created directly via the constructor. They are created in batches by higher-level systems, such as an objective manager, through the static factory methods `wrap` or `append`. These methods consume an EventRegistry and produce an array of records representing the active registrations within it.
-   **Scope:** The lifecycle of a RegistrationTransactionRecord is strictly bound to the lifecycle of the parent transaction it is part of. It is a short-lived object, existing only for the duration of a specific, stateful game logic operation.
-   **Destruction:** The object is eligible for garbage collection once the parent transaction is resolved (via `revert`, `complete`, or `unload`). The invocation of these methods triggers the record's final action—the un-registration callback—after which it serves no further purpose.

## Internal State & Concurrency
-   **State:** The internal state consists of a single `registration` field of type BooleanConsumer. This field is set at construction and is never modified, making the object's internal reference effectively immutable. The record itself is stateless; its purpose is to mutate the state of an external system (the EventRegistry).
-   **Thread Safety:** This class is **not thread-safe**. It contains no synchronization primitives and is designed to be created, managed, and executed on the main game logic thread. All interactions with the transaction and event systems are assumed to occur in a single-threaded context. Off-thread modification would introduce severe race conditions in the underlying EventRegistry.

## API Surface
The public API is focused on fulfilling the TransactionRecord contract and providing static factories for instantiation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| revert() | void | O(1) | Reverts the transaction by executing the un-registration callback. |
| complete() | void | O(1) | Completes the transaction by executing the un-registration callback. |
| unload() | void | O(1) | Cleans up during an unload sequence by executing the un-registration callback. |
| shouldBeSerialized() | boolean | O(1) | **Critical:** Always returns false to prevent persistence. |
| wrap(EventRegistry) | static TransactionRecord[] | O(N) | Factory to convert all registrations in a registry into an array of records. |
| append(TransactionRecord[], EventRegistry) | static TransactionRecord[] | O(N+M) | Factory to append new registration records to an existing array. |

## Integration Patterns

### Standard Usage
The intended use is for a managing system to capture the state of temporary event listeners within a transaction. This guarantees cleanup if the operation is rolled back.

```java
// An objective manager prepares a new quest step that requires temporary listeners.
EventRegistry temporaryListeners = objective.getTemporaryEventRegistry();
// ... game logic registers listeners for the step into temporaryListeners ...

// Wrap the new registrations in transaction records for rollback safety.
TransactionRecord[] records = RegistrationTransactionRecord.wrap(temporaryListeners);

// Add these records to the main transaction for this game operation.
masterTransaction.addRecords(records);

// If the quest step is later cancelled or fails:
masterTransaction.revert(); // This automatically calls revert() on each record, cleaning up the listeners.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new RegistrationTransactionRecord()`. This bypasses the intended creation flow from an EventRegistry and can lead to records that do not correspond to any real registration. Always use the static `wrap` or `append` methods.
-   **State Leakage:** Do not retain references to RegistrationTransactionRecord instances after their parent transaction has been resolved. They are ephemeral and should be allowed to be garbage collected.
-   **Asynchronous Execution:** Do not invoke `revert` or `complete` from a different thread than the one that manages the EventRegistry. This will break thread-safety assumptions of the event system.

## Data Pipeline
The component acts as a control-flow mechanism rather than a data-processing one. Its flow is centered on capturing registration state and later triggering an action based on that state.

> Flow:
> EventRegistry State -> `RegistrationTransactionRecord.wrap()` -> **RegistrationTransactionRecord** Array -> Transaction Manager -> `revert()`/`complete()` Call -> Unregister Callback Execution -> Modified EventRegistry State

