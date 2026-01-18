---
description: Architectural reference for Label
---

# Label

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.operation
**Type:** Transient

## Definition
```java
// Signature
public class Label {
```

## Architecture & Concepts
The Label class represents a strongly-typed destination marker within a sequence of server-side operations. It functions analogously to a label in assembly language or a `GOTO` target, providing a reference point for control-flow instructions like conditional jumps.

Its primary architectural purpose is to replace the use of raw integer indices for jump targets, preventing a class of "magic number" bugs and improving the readability of complex interaction logic. A Label is not a user-facing UI element; it is a purely programmatic construct for the server's interaction execution engine.

The constructor for Label is `protected`, which is a critical design constraint. This intentionally prevents arbitrary instantiation from outside its package. Labels are meant to be created and managed exclusively by a higher-level system, such as an `OperationBuilder` or a state machine compiler, which guarantees the integrity and uniqueness of the label's index within a given operational context.

## Lifecycle & Ownership
- **Creation:** A Label is not instantiated directly by client code. It is created and returned by a factory or builder class within the `com.hypixel.hytale.server.core.modules.interaction.interaction.operation` package. This factory is responsible for assigning a valid, context-specific integer index.
- **Scope:** The lifetime of a Label is ephemeral and is strictly bound to the operation list or interaction script in which it was defined. It holds no global state and becomes irrelevant once its parent interaction is executed or discarded.
- **Destruction:** The object is managed by the Java garbage collector. No manual cleanup is required. It is reclaimed once it is no longer referenced by any pending operation list.

## Internal State & Concurrency
- **State:** The Label's state is effectively immutable. The internal `index` is set once at construction and cannot be modified through its public API. This makes the object a simple and predictable data carrier.
- **Thread Safety:** This class is inherently thread-safe. As an immutable data holder, it can be safely read by multiple threads without any external synchronization. The atomicity of reading the primitive `int` index is guaranteed by the Java Memory Model.

## API Surface
The public API is minimal, exposing only the essential data required to identify the label.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getIndex() | int | O(1) | Returns the zero-based numerical index for this label. This index is only meaningful within the context of the operation list that created it. |

## Integration Patterns

### Standard Usage
A developer will typically receive a Label instance from a builder or factory method when defining a sequence of operations. It is then used as an argument for control-flow operations.

```java
// A hypothetical builder is responsible for creating and managing Labels
OperationBuilder builder = new OperationBuilder();

// 1. A new Label is created and registered by the builder
Label endOfLogicLabel = builder.createLabel();

// 2. The Label is used as a target for a jump operation
builder.add(new JumpIfTrueOperation(someCondition, endOfLogicLabel));
builder.add(new PerformActionOperation(actionA));

// 3. The builder uses the mark() method to associate the Label's
//    index with the current position in the operation list.
builder.mark(endOfLogicLabel);
builder.add(new PerformActionOperation(actionB));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The `protected` constructor forbids this from outside the package. Do not attempt to subclass Label merely to expose a public constructor, as this breaks the core design assumption that indices are managed by a central authority.
- **Index Re-use:** Do not retrieve an index from one Label and assume it is valid in the context of a different interaction script. Label indices are not globally unique.

## Data Pipeline
A Label does not process data itself; instead, it acts as a routing instruction within a data pipeline. It directs the flow of execution within an operation interpreter.

> Flow:
> Operation List Builder -> **Label** (Creation as a placeholder) -> Jump Operation (Stores a reference to the Label) -> Operation Executor (Resolves Label to an instruction index) -> Execution Pointer Update

