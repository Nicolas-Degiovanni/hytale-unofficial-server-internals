---
description: Architectural reference for TransactionRecord
---

# TransactionRecord

**Package:** com.hypixel.hytale.builtin.adventure.objectives.transaction
**Type:** Data Model (Abstract Base)

## Definition
```java
// Signature
public abstract class TransactionRecord {
```

## Architecture & Concepts
The TransactionRecord class is the abstract foundation for a command pattern implementation within Hytale's adventure mode objective system. It represents a single, reversible, and serializable operation that modifies the game world state. This system provides atomicity and durability for complex quest-related events.

Its primary architectural role is to decouple the execution of a game state change from the logic that initiated it. For example, when an objective needs to spawn a unique entity, it creates a SpawnEntityTransactionRecord. This record encapsulates all information required to perform, revert, or finalize the spawn. A collection of these records can be managed as a single unit of work, allowing an entire multi-step quest event to be rolled back if one part fails.

The static CODEC field, a CodecMapCodec, is central to its design. This enables polymorphic serialization and deserialization of different transaction types. The engine can save a list of heterogeneous TransactionRecord objects to disk and perfectly reconstruct them on load, a cornerstone of Hytale's data-driven architecture.

### Lifecycle & Ownership
-   **Creation:** Concrete subclasses (e.g., SpawnEntityTransactionRecord) are instantiated by higher-level game logic, typically an objective or quest script, in response to a world-altering event. The base abstract class is never instantiated directly.
-   **Scope:** A TransactionRecord's lifetime is bound to the objective or game event that created it. It persists as long as the operation it represents is considered reversible or incomplete. Once committed via the complete method, it may be discarded.
-   **Destruction:** The object is eligible for garbage collection when its owning objective is completed or unloaded. The unload method provides a hook for explicitly releasing any held references to live game world objects (like entities or blocks) to prevent memory leaks when a world chunk is unloaded.

## Internal State & Concurrency
-   **State:** Highly mutable. The internal state, including the TransactionStatus and a failure reason, is designed to be modified post-instantiation. Subclasses will hold mutable references to game world objects.
-   **Thread Safety:** This class is **not thread-safe** and is designed to be accessed exclusively from the main server or client game thread. Modifying game state via revert or complete from any other thread will lead to severe concurrency issues, race conditions, and world corruption. All interactions must be synchronized with the primary game loop.

## API Surface
The public API defines the contract for managing a transactional operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| revert() | abstract void | Varies | Reverses the effects of the transaction on the game world. |
| complete() | abstract void | Varies | Finalizes the transaction, making its effects permanent. |
| unload() | abstract void | Varies | Releases references to any loaded game world objects. |
| shouldBeSerialized() | abstract boolean | O(1) | Determines if this record needs to be persisted in a save file. |
| fail(String reason) | TransactionRecord | O(1) | Marks the transaction as FAILED and records a reason. |
| getStatus() | TransactionStatus | O(1) | Returns the current status of the transaction. |
| appendTransaction(...) | static TransactionRecord[] | O(N) | Utility to append a new transaction to an existing array. |
| appendFailedTransaction(...) | static TransactionRecord[] | O(N) | Utility to fail and append a new transaction in one call. |

## Integration Patterns

### Standard Usage
A managing system, such as a quest, accumulates TransactionRecord objects as it performs actions. To roll back the entire quest step, it iterates the list and calls revert on each record.

```java
// An objective accumulates transactions
TransactionRecord[] transactions = null;

// Spawning an entity
SpawnEntityTransactionRecord spawnTx = new SpawnEntityTransactionRecord(...);
// Execute the spawn
spawnTx.performSpawn(); 
transactions = TransactionRecord.appendTransaction(transactions, spawnTx);

// Later, if a failure occurs...
if (failureCondition) {
    for (TransactionRecord tx : transactions) {
        tx.revert();
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new TransactionRecord()`. This is an abstract class and will result in a compilation error. Always instantiate a concrete subclass.
-   **Asynchronous Execution:** Do not call `revert()` or `complete()` from a separate thread. All state modifications must occur on the main game thread.
-   **State Ambiguity:** Avoid calling `revert()` on a transaction that has already been completed, or vice-versa. The lifecycle methods are typically terminal operations.
-   **Ignoring Static Appenders:** Manually managing the transaction array is error-prone. Use the provided static helpers `appendTransaction` and `appendFailedTransaction` for correctness.

## Data Pipeline
TransactionRecord is a critical component in the game state persistence pipeline, ensuring that adventure mode progress is saved and loaded reliably.

> **Serialization Flow:**
> Active Quest State -> **TransactionRecord[]** -> `CODEC.encode()` -> Hytale Binary Format -> World Save File

> **Deserialization Flow:**
> World Save File -> Hytale Binary Format -> `CODEC.decode()` -> **TransactionRecord[]** -> Quest State Restoration -> World State Reconciliation

