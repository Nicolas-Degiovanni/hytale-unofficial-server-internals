---
description: Architectural reference for JumpOperation
---

# JumpOperation

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.operation
**Type:** Transient Command Object

## Definition
```java
// Signature
public class JumpOperation implements Operation {
```

## Architecture & Concepts
The JumpOperation is a fundamental control flow primitive within the server-side Interaction system. It functions as an unconditional jump, analogous to a GOTO statement or a JMP instruction in assembly language. Its sole purpose is to alter the execution sequence of an interaction script.

Interaction scripts are defined as a linear sequence of Operation objects. An executor, likely part of a higher-level interaction module, processes these operations sequentially. When the executor encounters a JumpOperation, it does not proceed to the next operation in the sequence. Instead, it uses the target Label within the JumpOperation to set the execution pointer (the *operationCounter* in the InteractionContext) to a new, non-sequential index.

This class is a key enabler for creating complex, non-linear interaction behaviors such as loops, conditional branches (when combined with other operations), and state-based forks without requiring a more complex scripting language. It is a stateless, immutable command whose execution atomically modifies the state of a parent InteractionContext.

### Lifecycle & Ownership
- **Creation:** JumpOperation instances are not created dynamically during gameplay. They are instantiated once during server startup or when game configuration is loaded. A parser or builder class within the `operation` package is responsible for reading a script definition (e.g., from a JSON or HOCON file) and constructing the entire sequence of Operations, including any JumpOperation objects. The `protected` constructor enforces this pattern.
- **Scope:** An instance of JumpOperation persists for the entire server lifetime, or until a configuration reload occurs. It is a shared, immutable object referenced by one or more interaction script definitions.
- **Destruction:** The object is eligible for garbage collection only when the script definition that contains it is unloaded.

## Internal State & Concurrency
- **State:** The JumpOperation is **immutable**. Its internal state, the `target` Label, is final and set at construction time. It holds no mutable state and does not cache any data. All stateful changes are performed on the external InteractionContext object passed into its methods.
- **Thread Safety:** This class is inherently **thread-safe**. Because it is immutable, a single JumpOperation instance can be safely referenced and executed by multiple threads simultaneously, provided each execution operates on a distinct InteractionContext instance. The atomicity of its logic (two sequential setter calls) prevents internal race conditions.

## API Surface
The public contract is defined by the Operation interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(...) | void | O(1) | Executes the jump. Atomically sets the operation counter on the context and marks the operation as finished. |
| simulateTick(...) | void | O(1) | Performs the identical logic as tick. Used for predictive or speculative execution paths. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns `WaitForDataFrom.None`, indicating this is a synchronous, non-blocking operation. |

## Integration Patterns

### Standard Usage
The JumpOperation is executed by a state machine or processor that iterates over an array of Operation objects. The processor maintains the current execution index and passes the relevant context to the current operation's `tick` method.

```java
// Conceptual executor loop
// operations is an array of Operation objects for a given script
// context is the state for a specific entity's interaction

int currentIndex = context.getOperationCounter();
Operation currentOp = operations[currentIndex];

// When currentOp is a JumpOperation instance:
currentOp.tick(..., context, ...);

// The tick method changes the counter inside the context.
// The loop's next iteration will fetch from the new index.
int nextIndex = context.getOperationCounter(); // This will now be the jump target's index
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create instances of JumpOperation via reflection. They are designed to be created exclusively by the framework's script parser to ensure the integrity of the `target` Label.
- **Incorrect Scripting:** The primary source of errors will be in the script definition itself. A JumpOperation pointing to its own index or a previous index without a proper exit condition will create an infinite loop, consuming server resources for that entity's interaction.
- **Out-of-Bounds Jumps:** A script configured with a JumpOperation whose target index is outside the bounds of the operation array will cause a runtime `IndexOutOfBoundsException` in the executor.

## Data Pipeline
JumpOperation does not process or transform data. Instead, it redirects the control flow of its parent executor.

> Flow:
> Interaction Executor reads `operationCounter` from `InteractionContext` -> Fetches **JumpOperation** from script array -> Executor calls `tick` -> **JumpOperation** writes new index to `InteractionContext.operationCounter` -> Executor loop continues from new index.

