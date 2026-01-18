---
description: Architectural reference for KillTaskTransaction
---

# KillTaskTransaction

**Package:** com.hypixel.hytale.builtin.adventure.npcobjectives.transaction
**Type:** Transient

## Definition
```java
// Signature
public class KillTaskTransaction extends TransactionRecord {
```

## Architecture & Concepts
The KillTaskTransaction class is a stateful, short-lived object that represents an active "kill tracking" objective within the server's adventure mode system. It acts as a transactional record, bridging a specific KillTask with the global KillTrackerResource.

Its primary architectural role is to manage the subscription lifecycle for a single objective. When a player activates a quest that requires killing a certain number of entities, an instance of KillTaskTransaction is created. This instance then registers itself with the KillTrackerResource, effectively subscribing to entity death events that match the objective's criteria.

This class embodies the **Observer Pattern**. The KillTaskTransaction is the handle to the observation, and the KillTrackerResource is the subject. By encapsulating the subscription logic, it decouples the high-level objective system from the low-level resource and event tracking mechanisms. The `shouldBeSerialized` method returning false is a critical design choice, indicating this is a pure runtime construct that must be rebuilt from persistent objective state upon server or world load.

## Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level objective management service when a KillTask becomes active for a player or entity. The creator is responsible for providing the specific task, its parent objective, and a ComponentAccessor to the relevant world or entity store.
- **Scope:** The object's lifetime is strictly bound to the duration of the active KillTask it represents. It is created when the task begins and is destroyed when the task is completed, failed, or unloaded.
- **Destruction:** The transaction is terminated and becomes eligible for garbage collection when `revert`, `complete`, or `unload` is called. Each of these methods performs the crucial cleanup step of calling `unwatch` on the KillTrackerResource, which removes the strong reference from the resource back to this transaction object.

**WARNING:** Failure to call one of the terminal lifecycle methods (`revert`, `complete`, `unload`) will result in a memory leak. The KillTrackerResource will retain a reference to the transaction, preventing it from being garbage collected.

## Internal State & Concurrency
- **State:** The internal fields of this class are final and assigned at construction, making the object's direct state **immutable**. However, its existence represents a mutable state within the broader objective system—specifically, an active subscription to the KillTrackerResource. It does not cache any data.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, accessed, and destroyed exclusively on the main server thread. All interactions with the ComponentAccessor and the underlying KillTrackerResource must be synchronized with the server's primary game loop. Unsynchronized access from other threads will lead to race conditions and unpredictable behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| revert() | void | O(1) | Terminates the transaction due to failure or cancellation. Deregisters from the KillTrackerResource. |
| complete() | void | O(1) | Terminates the transaction due to successful completion. Deregisters from the KillTrackerResource. |
| unload() | void | O(1) | Terminates the transaction because the associated entity or world is unloading. Deregisters from the KillTrackerResource. |
| getTask() | KillTask | O(1) | Returns the specific kill task this transaction is managing. |
| getObjective() | Objective | O(1) | Returns the parent objective that owns the associated task. |
| shouldBeSerialized() | boolean | O(1) | Returns false, indicating this is a runtime-only object not intended for persistence. |

## Integration Patterns

### Standard Usage
This object should be created and managed by a service that orchestrates objective progress. The service retains the instance and calls a terminal method based on game events.

```java
// Pseudo-code for an objective management service
ComponentAccessor<EntityStore> accessor = ...;
KillTask task = objective.getActiveTask();

// When the task becomes active, create the transaction
KillTaskTransaction transaction = new KillTaskTransaction(task, objective, accessor);
activeTransactions.add(transaction);

// When the objective is completed or cancelled later...
transaction.complete(); // or transaction.revert();
activeTransactions.remove(transaction);
```

### Anti-Patterns (Do NOT do this)
- **Leaking References:** Do not create a KillTaskTransaction and lose the reference to it without calling `complete`, `revert`, or `unload`. This will permanently leave the object registered in the KillTrackerResource, causing a memory leak.
- **Direct Instantiation without Management:** Do not instantiate this class without a higher-level system to manage its lifecycle. It is not a standalone component.
- **Cross-Thread Access:** Do not share instances of this class across threads or invoke its methods from asynchronous tasks. All interactions must occur on the main server thread.

## Data Pipeline
KillTaskTransaction does not process a continuous flow of data. Instead, it acts as a control object that initiates and terminates a listening state.

> **Activation Flow:**
> ObjectiveManager -> `new KillTaskTransaction(...)` -> **KillTaskTransaction** -> Registers with KillTrackerResource
>
> **Event Notification (while active):**
> Entity Death Event -> Server Engine -> KillTrackerResource -> (Internal logic notifies relevant objectives)
>
> **Deactivation Flow:**
> ObjectiveManager -> `transaction.complete()` -> **KillTaskTransaction** -> Deregisters from KillTrackerResource

