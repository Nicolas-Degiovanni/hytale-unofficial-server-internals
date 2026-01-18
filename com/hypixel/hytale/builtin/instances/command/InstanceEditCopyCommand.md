---
description: Architectural reference for InstanceEditCopyCommand
---

# InstanceEditCopyCommand

**Package:** com.hypixel.hytale.builtin.instances.command
**Type:** Transient

## Definition
```java
// Signature
public class InstanceEditCopyCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
The InstanceEditCopyCommand is a concrete implementation of the Command Pattern, designed to handle a specific administrative task: duplicating a world instance on the server's filesystem. It functions as a self-contained unit of work within the server's command processing system.

Architecturally, its most critical feature is its extension of AbstractAsyncCommand. This signals to the server's Command Dispatcher that the operation is potentially long-running and involves blocking I/O. Consequently, the dispatcher executes this command on a dedicated worker thread pool, preventing the main server game loop from stalling while large directories are being copied.

This class encapsulates all logic for this operation, including:
1.  **Argument Parsing:** Defines the required command arguments (origin and destination names).
2.  **Pre-condition Validation:** Performs crucial safety checks, such as verifying the immutability of the base asset pack and ensuring the source instance exists and the destination does not.
3.  **Execution Logic:** Orchestrates the file copy operation, loads the source world configuration, generates a new unique identifier (UUID), and saves the new configuration.
4.  **User Feedback:** Reports success or detailed failure messages back to the command issuer via the CommandContext.

It serves as a bridge between the user-facing command system and the low-level filesystem and world configuration services.

### Lifecycle & Ownership
-   **Creation:** A single prototype instance of InstanceEditCopyCommand is instantiated by the Command System during server or plugin initialization. This happens via reflection or direct registration. It is not created on a per-request basis.
-   **Scope:** The prototype instance is a long-lived object, persisting for the entire server session within the central command registry. However, the execution itself is scoped to the lifecycle of the `executeAsync` method call.
-   **Destruction:** The prototype instance is discarded when the server shuts down or the parent plugin is disabled, at which point the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is fundamentally stateless. The member fields `originNameArg` and `destinationNameArg` are final and define the command's structure, not its execution state. All state required for an operation (e.g., the specific instance names) is provided externally via the CommandContext argument in the `executeAsync` method.

-   **Thread Safety:** The class is inherently thread-safe. Because it is stateless, a single prototype instance can be safely used to execute commands originating from multiple sources concurrently. Each execution runs in its own self-contained `CompletableFuture` on a worker thread, with no shared mutable state between invocations. No external locking mechanisms are required.

## API Surface
The primary public contract is the `executeAsync` method, inherited from its parent class. The constructor is intended for framework use only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context) | CompletableFuture<Void> | O(N) | Asynchronously executes the instance copy logic. Complexity is proportional to the size (N) of the source instance directory on disk. This method performs all validation, I/O, and user feedback. |

## Integration Patterns

### Standard Usage
This class is not designed to be invoked directly. It is registered with the server's command system, which handles its lifecycle and invocation in response to user input from the console or an in-game client. A developer would typically register this command within a plugin's entry point.

```java
// Conceptual registration in a plugin
// The framework handles the invocation.
CommandSystem commandSystem = server.getCommandSystem();
commandSystem.register(new InstanceEditCopyCommand());

// A user then executes it via text input:
// /instance edit copy "my_template_world" "new_world_copy"
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation and Invocation:** Do not create an instance and call `executeAsync` manually. This bypasses the entire command processing pipeline, including permission checks, argument parsing from source, and asynchronous execution management.

-   **Blocking on the Main Thread:** Calling `.join()` or `.get()` on the `CompletableFuture` returned by this command from the main server thread is a critical error. It will freeze the server and completely defeat the purpose of using an `AbstractAsyncCommand`.

-   **Stateful Modifications:** Do not add mutable member fields to this class. It is designed as a stateless singleton; adding state will introduce severe concurrency bugs.

## Data Pipeline
The flow of data for this command begins with user input and ends with a filesystem modification and a feedback message. The command class itself is the central processing unit in this flow.

> Flow:
> User Input (`/instance edit copy...`) -> Network Layer -> Command Parser -> Command Dispatcher -> **InstanceEditCopyCommand.executeAsync** -> Filesystem Read (origin) -> BSON Deserialization -> State Modification (new UUID) -> BSON Serialization -> Filesystem Write (destination) -> Asynchronous Task Completion -> CommandContext Feedback -> Network Layer -> User Console/Chat

---

