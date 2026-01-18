---
description: Architectural reference for SpawnTreasureChestTransactionRecord
---

# SpawnTreasureChestTransactionRecord

**Package:** com.hypixel.hytale.builtin.adventure.objectives.transaction
**Type:** Transient Data Object

## Definition
```java
// Signature
public class SpawnTreasureChestTransactionRecord extends TransactionRecord {
```

## Architecture & Concepts

The SpawnTreasureChestTransactionRecord is a concrete implementation of the Command design pattern, encapsulated within the server's transactional framework. It represents a single, reversible action: the spawning of a treasure chest within a specific world.

This class is not an entity or a service; it is a data-driven instruction. Its primary role is to provide the logic for *reverting* a game state change. The presence of a static CODEC field indicates that this instruction is designed to be serialized to binary format, allowing it to be persisted in save files or transmitted over the network. This is fundamental to ensuring that long-running or multi-stage objectives can survive server restarts and maintain state integrity.

The core responsibility of this record is to hold the coordinates (`worldUUID`, `blockPosition`) of the spawned chest. The `revert` method contains the logic to undo the spawn, which, in this implementation, involves setting the chest's block state to "opened" rather than removing the block entirely. This suggests the transaction system is used for managing logical game state, not just physical world modifications.

## Lifecycle & Ownership

-   **Creation:** An instance is created by a higher-level game system, such as an objective or quest manager, when a quest requires a treasure chest to be placed in the world. It is immediately submitted to a transaction management service.
-   **Scope:** The object's lifetime is tied to the parent transaction. It persists as long as the transaction is active, potentially across server sessions if serialized.
-   **Destruction:** The object is eligible for garbage collection once the owning transaction is fully committed or reverted and is no longer held in memory by the transaction manager. The `unload` method, while empty, marks a point in the lifecycle where external resources would typically be released.

## Internal State & Concurrency

-   **State:** The internal state, consisting of `worldUUID` and `blockPosition`, is set during construction or deserialization and is not modified thereafter. The object is *effectively immutable* after its initial creation. It is a pure data container for the transaction parameters.
-   **Thread Safety:** This class is **not thread-safe**. While its internal fields are simple data types, the `revert` method interacts directly with the global `Universe` and `World` state. All methods that modify world state, such as `revert`, must be executed exclusively on the main server thread to prevent race conditions and world corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| revert() | void | O(1) | Reverts the transaction. Finds the chest at the stored position and sets its state to opened. **WARNING:** Must be called from the main server thread. |
| complete() | void | O(1) | A lifecycle hook for transaction completion. This implementation is a no-op. |
| unload() | void | O(1) | A lifecycle hook for when the transaction is unloaded from memory. This implementation is a no-op. |
| shouldBeSerialized() | boolean | O(1) | Returns true, indicating this record should be persisted with the game state. |

## Integration Patterns

### Standard Usage

This class is not intended for direct invocation by most game logic. It is designed to be created and managed by a transaction service. A system responsible for quests would create an instance and add it to a transaction, which then controls its lifecycle.

```java
// Pseudo-code demonstrating conceptual usage by a quest system
UUID worldId = ...;
Vector3i chestPosition = ...;

// The record is created as part of a larger transaction
Transaction transaction = new Transaction();
transaction.add(new SpawnTreasureChestTransactionRecord(worldId, chestPosition));

// The transaction is then managed by a central service
TransactionService.get().submit(transaction);
```

### Anti-Patterns (Do NOT do this)

-   **Manual Lifecycle Management:** Do not call `revert()` or `complete()` directly. These methods are exclusively for the owning transaction manager to invoke. Calling them manually will desynchronize game state.
-   **State Modification:** Do not attempt to modify the `worldUUID` or `blockPosition` fields after instantiation. This will lead to unpredictable behavior, especially if the transaction needs to be reverted.
-   **Cross-Thread Access:** Never call `revert()` from an asynchronous task or a different thread. All world modifications must be synchronized with the main server tick.

## Data Pipeline

The primary data flow for this class is as a command object that carries state and behavior through the transaction system.

> **Execution Flow:**
> Quest System Logic -> Instantiates **SpawnTreasureChestTransactionRecord** -> Transaction Manager -> (On Revert Signal) -> `revert()` -> Universe -> World -> WorldChunk -> BlockState Update

> **Serialization Flow:**
> Game Save Event -> Transaction Manager -> **SpawnTreasureChestTransactionRecord** -> `CODEC` -> Binary Data -> Disk / Network Stream

