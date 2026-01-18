---
description: Architectural reference for TransactionUtil
---

# TransactionUtil

**Package:** com.hypixel.hytale.builtin.adventure.objectives.transaction
**Type:** Utility

## Definition
```java
// Signature
public class TransactionUtil {
```

## Architecture & Concepts
TransactionUtil is a stateless, static utility class that provides bulk operations for managing collections of TransactionRecord objects. It resides within the Adventure Mode objective system and serves as a foundational component for ensuring transactional integrity across multi-step game objectives.

In Hytale's objective system, a single logical action (e.g., "Craft a Sword") may involve multiple underlying state changes (e.g., remove ore from inventory, remove coal, add sword to inventory). Each of these state changes is encapsulated in a TransactionRecord. TransactionUtil provides the coarse-grained control logic to treat these individual records as a single atomic unit. Its primary function is to centralize the iteration and lifecycle management of these collections, allowing higher-level systems like an ObjectiveManager to commit, roll back, or query the status of an entire operation with a single method call.

This class enforces a consistent pattern for handling objective state changes, preventing boilerplate code and reducing the risk of partial updates that could lead to corrupted game states.

### Lifecycle & Ownership
- **Creation:** As a static utility class, TransactionUtil is never instantiated. Its methods are accessed directly from the class type. The Java Virtual Machine loads the class definition into memory on first access.
- **Scope:** The class and its static methods are globally accessible throughout the application's lifetime, subject to standard Java classloader visibility rules. It has no instance-level scope.
- **Destruction:** The class definition is unloaded when the corresponding ClassLoader is garbage collected, which typically occurs during application shutdown. There are no instances to manage or destroy.

## Internal State & Concurrency
- **State:** TransactionUtil is completely stateless. It contains no member variables and its methods operate exclusively on the arguments provided to them. Its behavior is deterministic and functional.
- **Thread Safety:** This class is inherently thread-safe. Since it has no internal state, calls from multiple threads cannot interfere with each other.
    - **Warning:** While the utility itself is thread-safe, the collections of TransactionRecord objects passed to it may not be. The caller is responsible for ensuring that the underlying TransactionRecord array is not being mutated by another thread while a TransactionUtil method is iterating over it. Failure to do so can result in concurrency exceptions or unpredictable behavior within the TransactionRecord instances themselves.

## API Surface
The public API consists of static helper methods for operating on arrays of TransactionRecord.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| anyFailed(TransactionRecord[]) | boolean | O(N) | Iterates through the records, returning true if any have a status of FAIL. Short-circuits on the first failure. |
| revertAll(TransactionRecord[]) | void | O(N) | Invokes the revert method on every record in the array. This is a critical rollback operation. |
| completeAll(TransactionRecord[]) | void | O(N) | Invokes the complete method on every record in the array. This is the final commit operation. |
| unloadAll(TransactionRecord[]) | void | O(N) | Invokes the unload method on every record, typically for resource cleanup after the transaction is finished. |

## Integration Patterns

### Standard Usage
The canonical use case is within a higher-level service that orchestrates a complex game state change. The service first attempts an operation, which yields a set of transaction records, and then uses TransactionUtil to determine whether to commit or roll back the entire operation.

```java
// An ObjectiveManager attempts to execute a multi-step task
TransactionRecord[] records = objective.executeTask();

// Use the utility to check the result of the entire operation
if (TransactionUtil.anyFailed(records)) {
    // At least one step failed. Revert all changes to ensure consistency.
    TransactionUtil.revertAll(records);
    log.error("Objective task failed, state has been rolled back.");
} else {
    // All steps succeeded. Commit the changes permanently.
    TransactionUtil.completeAll(records);
    log.info("Objective task completed successfully.");
}

// Finally, unload resources associated with the transaction
TransactionUtil.unloadAll(records);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Attempting to create an instance via `new TransactionUtil()` is not possible and indicates a misunderstanding of the class's purpose. All methods are static.
- **Ignoring Failure States:** A common and critical error is to call `completeAll` without first checking for failures with `anyFailed`. This can leave the game in an inconsistent state, where a failed operation is incorrectly marked as complete.

    ```java
    // BAD: This code commits a potentially failed transaction
    TransactionRecord[] records = objective.executeTask();
    TransactionUtil.completeAll(records); // This may commit a partial, broken state
    ```

## Data Pipeline
TransactionUtil does not participate in a data pipeline; rather, it acts as a control-flow mechanism for processing batches of state-change objects. It is a terminal processor or decision point in a larger workflow.

> Flow:
> ObjectiveManager::execute() → `TransactionRecord[]` → **TransactionUtil::anyFailed()** → [Decision] → **TransactionUtil::revertAll()** / **TransactionUtil::completeAll()** → Game State Update

