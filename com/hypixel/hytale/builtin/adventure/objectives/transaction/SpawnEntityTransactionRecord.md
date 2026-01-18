---
description: Architectural reference for SpawnEntityTransactionRecord
---

# SpawnEntityTransactionRecord

**Package:** com.hypixel.hytale.builtin.adventure.objectives.transaction
**Type:** Data Record

## Definition
```java
// Signature
public class SpawnEntityTransactionRecord extends TransactionRecord {
```

## Architecture & Concepts
The SpawnEntityTransactionRecord is a concrete implementation of the Command pattern, designed to operate within a transactional system. Its primary role is to encapsulate a single, reversible action: the spawning of an entity. This class does not perform the spawn itself; rather, it holds the necessary information—the World and Entity UUIDs—to *undo* the spawn at a later time.

This component is critical for systems that create temporary or conditional game state, such as adventure mode objectives or quests. When a quest requires a temporary NPC to be spawned, a corresponding SpawnEntityTransactionRecord is created and registered with a transaction manager. This ensures that if the quest is failed, abandoned, or even successfully completed, the temporary entity can be reliably cleaned up, preventing world state corruption and entity leaks.

The presence of a static CODEC field signifies that these records are designed for persistence. The server can serialize the state of an in-progress transaction, including this record, allowing complex, multi-step objectives to survive server restarts and be correctly resolved later.

## Lifecycle & Ownership
-   **Creation:** An instance is created by a higher-level system, such as an Objective or Quest manager, immediately after it has spawned an entity whose existence is conditional. The record is then added to a parent Transaction object.
-   **Scope:** The object's lifetime is strictly bound to its parent transaction. It persists as long as the transaction is considered "in-flight" or requires the possibility of a rollback.
-   **Destruction:** The object becomes eligible for garbage collection once the parent transaction manager invokes its `revert` or `complete` method and drops its reference. It is a short-lived object by design.

## Internal State & Concurrency
-   **State:** The internal state, consisting of `worldUUID` and `entityUUID`, is effectively immutable after construction. The object acts as a historical record of an event and its state should not be altered.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be created, managed, and executed exclusively on the main server thread. Its core logic in `removeEntity` directly accesses and modifies global server state (Universe and World), which is an unsafe operation to perform from any other thread. All interactions with this class must be synchronized with the server's main game loop.

## API Surface
The public API is minimal, exposing only the methods required to fulfill its contract with the transaction management system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| revert() | void | O(1) | Reverses the transaction by removing the associated entity from the world. Throws a RuntimeException if the entity cannot be found or removed. |
| complete() | void | O(1) | Finalizes the transaction by removing the associated entity. This behavior implies the entity was temporary and must be cleaned up on successful completion. |
| shouldBeSerialized() | boolean | O(1) | A flag indicating that this record should be persisted with its parent transaction. Always returns true. |

## Integration Patterns

### Standard Usage
This record should never be used in isolation. It must be created as part of a larger transaction managed by a quest or objective system.

```java
// PSEUDO-CODE: Illustrates the intended integration with a transaction manager.

// 1. A system spawns a temporary entity.
Entity temporaryNpc = world.spawnEntity(npcData);

// 2. A record is created to track this action.
TransactionRecord record = new SpawnEntityTransactionRecord(
    world.getUUID(),
    temporaryNpc.getUUID()
);

// 3. The record is added to the current transaction.
// The transactionManager is responsible for calling revert() or complete() later.
transactionManager.addRecord(record);
```

### Anti-Patterns (Do NOT do this)
-   **Manual Invocation:** Never call `revert` or `complete` directly. These methods are callbacks intended exclusively for the parent transaction manager. Invoking them manually will break the transactional integrity of the system.
-   **State Modification:** Do not attempt to modify the `worldUUID` or `entityUUID` fields after instantiation. The record is a historical fact and must remain unaltered.
-   **Orphaned Instances:** Do not create an instance of this class without registering it to a transaction manager. An orphaned record serves no purpose and will not perform its cleanup duties.

## Data Pipeline
This component acts as a final step in a control flow, executing a side-effect (entity removal) based on instructions from a transaction manager. It does not transform data.

> **Control Flow:**
> Quest Logic spawns an Entity → **SpawnEntityTransactionRecord is created** with Entity/World UUIDs → Record is stored by a Transaction Manager → (On transaction rollback or completion) → Manager calls `revert()` or `complete()` → **SpawnEntityTransactionRecord** uses UUIDs to find and remove the Entity from the World.

