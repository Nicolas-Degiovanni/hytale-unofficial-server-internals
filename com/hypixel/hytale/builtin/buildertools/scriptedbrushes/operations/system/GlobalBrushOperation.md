---
description: Architectural reference for GlobalBrushOperation
---

# GlobalBrushOperation

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.system
**Type:** Abstract Template

## Definition
```java
// Signature
public abstract class GlobalBrushOperation extends BrushOperation {
```

## Architecture & Concepts
GlobalBrushOperation is an abstract base class that serves as a foundational template for any scripted brush operation intended to affect a large, non-localized area of the game world. It extends the base BrushOperation, inheriting its core contract, but provides a specific categorization for operations that are not bound to a player's immediate vicinity or a small, targeted volume.

Architecturally, this class acts as a marker and a contract within the Builder Tools framework. The system uses this type to differentiate between localized brushes (e.g., painting a small sphere of blocks) and global modifications (e.g., replacing all instances of one block type with another across the entire world). This distinction allows the engine to apply different validation rules, performance considerations, and transaction logic.

Subclasses of GlobalBrushOperation represent a single, atomic world modification command. They encapsulate the logic required to perform a large-scale change, but do not execute it themselves. The execution is orchestrated by the core Brush System, which manages the world state and ensures operational integrity.

## Lifecycle & Ownership
- **Creation:** This abstract class is never instantiated directly. Concrete subclasses are instantiated by the Scripted Brush System or a command parser when a user or script invokes a global build tool action. The instantiation is typically just-in-time for the operation's execution.

- **Scope:** An instance of a GlobalBrushOperation subclass is extremely short-lived. Its lifecycle is bound to a single user-initiated action. It is created, its parameters are configured, it is passed to the Brush System for execution, and then it is immediately eligible for garbage collection.

- **Destruction:** The object is discarded and garbage collected as soon as the world modification it represents has been completed or has failed. There is no mechanism for reuse or pooling.

## Internal State & Concurrency
- **State:** The base GlobalBrushOperation holds no state beyond the `name` and `description` inherited from its parent. Concrete implementations are expected to be stateful during their configuration phase but should be treated as effectively immutable once submitted for execution. The state they carry represents the parameters of the command (e.g., target block type, replacement block type).

- **Thread Safety:** **WARNING:** Implementations of this class are fundamentally **not thread-safe** and must never be treated as such. All world modification logic must be executed exclusively on the main world thread to prevent catastrophic race conditions, chunk corruption, and server instability. The Brush System enforces this, but developers extending this class must not introduce asynchronous behavior.

## API Surface
As an abstract class, GlobalBrushOperation exposes no new public API. Its primary purpose is to enforce a design contract through inheritance. The relevant API is defined in the parent BrushOperation class, which concrete subclasses must implement.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| GlobalBrushOperation(name, description) | constructor | O(1) | Protected constructor for use by subclasses. Initializes the operation's metadata. |

## Integration Patterns

### Standard Usage
The standard pattern is to extend this class to define a new, custom global world editing command. The engine's Brush System will then discover and manage the execution of this new operation.

```java
// A concrete implementation of a global operation
public class ReplaceAllBlocksOperation extends GlobalBrushOperation {

    private final BlockType target;
    private final BlockType replacement;

    public ReplaceAllBlocksOperation(BlockType target, BlockType replacement) {
        super("Replace All", "Replaces all blocks of one type with another.");
        this.target = target;
        this.replacement = replacement;
    }

    // The core logic is implemented in an overridden method
    // from the BrushOperation parent class (e.g., applyToWorld).
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Execution:** Never attempt to invoke the operation's logic directly. Always submit the operation instance to the central Brush System or World Transaction Manager. Bypassing the system will lead to state desynchronization and world corruption.
- **Long-Lived Instances:** Do not hold references to operation instances after they have been executed. They are single-use command objects and are not designed to be reused.
- **Asynchronous Modification:** Do not dispatch world modification logic from a subclass to a background thread. All interactions with the world state must be synchronized with the main game tick.

## Data Pipeline
The data flow for a global brush operation is centrally managed and transactional to ensure world integrity.

> Flow:
> User Command or Script Execution -> Command Parser -> **(Subclass of) GlobalBrushOperation Instantiation & Configuration** -> Submission to Brush System -> World Transaction Queue -> Synchronous Execution on World State -> Replication to Clients

