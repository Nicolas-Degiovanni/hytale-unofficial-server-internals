---
description: Architectural reference for OpAddCommand
---

# OpAddCommand

**Package:** com.hypixel.hytale.server.core.permissions.commands.op
**Type:** Transient

## Definition
```java
// Signature
public class OpAddCommand extends CommandBase {
```

## Architecture & Concepts
The OpAddCommand class is a concrete implementation of the Command Pattern, designed to handle a specific server console or in-game administrative action: granting operator status to a player. It does not function as a long-lived service but as a single-purpose, transactional script.

Within the server architecture, this class resides at the intersection of the Command System and the Permissions System. When a user issues the `/op add` command, the server's central command dispatcher identifies and invokes this class to perform the necessary logic. Its sole responsibility is to parse the target player from the command context, interact with the `PermissionsModule` to elevate their privileges, and provide feedback to the relevant parties. It is a terminal component in a user-input-driven workflow, translating a high-level command into a low-level state change within a core server module.

### Lifecycle & Ownership
-   **Creation:** A single instance of OpAddCommand is instantiated by the server's command registration system during the bootstrap phase. The system typically scans for all classes extending CommandBase and registers them for the server's lifetime.
-   **Scope:** While the object instance persists for the entire server session, its execution is scoped to a single command invocation. It is stateless between calls, with all necessary information provided via the CommandContext parameter during execution.
-   **Destruction:** The instance is dereferenced and garbage collected when the server shuts down and the central command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is effectively **immutable** after its initial construction. Its fields, such as `playerArg` and the static `Message` templates, are final. It holds no mutable state related to any specific command execution; all runtime data is managed within the `CommandContext`.
-   **Thread Safety:** The `executeSync` method signature explicitly indicates that this class is **not thread-safe** and is designed to be executed synchronously on the main server thread. The server's command dispatcher guarantees this execution model. Any attempt to invoke this method from an asynchronous task or worker thread will result in race conditions and likely corrupt the state of the `PermissionsModule` or `Universe`.

## API Surface
The primary contract is the `executeSync` method, inherited from `CommandBase` and invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(log N) | Executes the command logic. Fetches the target player's UUID, modifies their group in the PermissionsModule, and dispatches feedback messages. Complexity is dependent on the underlying permission system's data structures. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. It is registered and invoked automatically by the server's command handling infrastructure. The interaction is initiated by a player or administrator typing the command into their client or the server console.

```
// User input that triggers this class
/op add Notch
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new OpAddCommand()`. The command system manages the lifecycle of all command objects. Manual creation will result in an unregistered command that is never executed.
-   **Manual Invocation:** Do not call `executeSync` directly. Bypassing the command dispatcher circumvents critical upstream processing, including permission checks, argument parsing, and context setup. This can lead to unpredictable behavior and security vulnerabilities.
-   **Asynchronous Execution:** Executing this command's logic off the main server thread is strictly forbidden. It performs non-thread-safe operations on core server modules like `PermissionsModule` and `Universe`, which will cause severe concurrency issues.

## Data Pipeline
The flow of data for this command begins with user input and terminates with a state change and network feedback.

> Flow:
> Player Command Input -> Network Layer -> Command Dispatcher -> Argument Parser -> **OpAddCommand** -> PermissionsModule -> Universe -> Message System -> Network Layer -> Player Client UI

