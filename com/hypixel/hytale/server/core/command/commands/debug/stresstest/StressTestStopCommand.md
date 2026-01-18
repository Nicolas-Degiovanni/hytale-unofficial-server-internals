---
description: Architectural reference for StressTestStopCommand
---

# StressTestStopCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug.stresstest
**Type:** Transient

## Definition
```java
// Signature
public class StressTestStopCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
The StressTestStopCommand is a command handler within the server's core Command System. Its sole responsibility is to provide the server-side logic for the administrative command that terminates a running stress test.

This class acts as a state transition trigger for a global, server-wide stress test. It does not manage the stress test lifecycle itself; it exclusively communicates a stop signal to the StressTestStartCommand class, which owns the state machine and control logic. This design demonstrates a clear separation of concerns between the command-line interface and the underlying system being controlled.

By extending AbstractAsyncCommand, it signals to the Command System that its execution should be handled by an asynchronous processing pipeline. This is a critical safety feature, ensuring that even if the stop logic were to become complex, it would not block the main server thread.

## Lifecycle & Ownership
- **Creation:** A prototype instance is created by the Command Registry during server bootstrap. The registry scans for command classes and instantiates them to map command strings to executable logic.
- **Scope:** The prototype instance persists for the entire server session, registered within the Command System. The execution itself is transient; a new context is created for each command invocation.
- **Destruction:** The instance is dereferenced and garbage collected when the server shuts down and the Command Registry is cleared.

## Internal State & Concurrency
- **State:** This class is **stateless**. It contains no mutable instance fields. All state it interacts with is a static, shared `AtomicReference` located in the StressTestStartCommand class. This design makes the command object itself immutable and reusable.
- **Thread Safety:** The class is inherently thread-safe. Its core logic is centered on the `compareAndSet` atomic operation on the shared state object. This is a lock-free, thread-safe mechanism that guarantees only one thread can successfully transition the stress test state from RUNNING to STOPPING, preventing critical race conditions.

## API Surface
The public contract is fulfilled by the implementation of the `executeAsync` method from its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context) | CompletableFuture<Void> | O(1) | Atomically attempts to stop the global stress test. Sends a feedback message to the command issuer. The operation is non-blocking. |

## Integration Patterns

### Standard Usage
A developer or administrator does not interact with this class directly in code. It is invoked exclusively by the server's Command System when a privileged user executes the corresponding command.

```java
// Conceptual representation of the Command System invoking the command.
// This code is illustrative and does not represent a direct API call.

// 1. A user types "/stresstest stop" into the console.
// 2. The server's CommandSystem parses the input.
// 3. It identifies "stresstest stop" and dispatches to the registered StressTestStopCommand instance.
CommandSystem commandSystem = server.getCommandSystem();
CommandContext context = createCommandContextForUser(user, "/stresstest stop");

// 4. The system invokes the command's logic asynchronously.
commandSystem.dispatch(context);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new StressTestStopCommand()`. The object is meaningless outside the context of the Command Registry, which is responsible for its lifecycle and invocation.
- **Incorrect State Assumption:** This command is tightly coupled to StressTestStartCommand. Invoking its logic without the corresponding start command and its state machine being properly initialized will result in predictable failure messages but indicates a flawed integration.

## Data Pipeline
The data flow for this component is initiated by user action and results in a system state change.

> Flow:
> User Input (`/stresstest stop`) -> Network Layer -> Command Parser -> **StressTestStopCommand.executeAsync()** -> Atomic State Transition -> Feedback Message -> Network Layer -> User Console

