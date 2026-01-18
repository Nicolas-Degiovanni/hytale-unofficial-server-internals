---
description: Architectural reference for WorldLoadCommand
---

# WorldLoadCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.world
**Type:** Transient

## Definition
```java
// Signature
public class WorldLoadCommand extends CommandBase {
```

## Architecture & Concepts
The WorldLoadCommand class is a component of the server's Command System. It serves as a user-facing entry point for the complex, asynchronous process of loading a game world from storage into active memory.

Architecturally, this class follows the **Command Pattern**. It encapsulates a request to load a world into a standalone object. Its primary responsibility is not to perform the loading itself, but to act as a transactional controller:
1.  **Define Arguments:** It declares the required arguments for the command, in this case, a world name.
2.  **Validate State:** It performs critical pre-flight checks against the central Universe service to prevent invalid operations, such as loading a world that is already active or one that does not exist.
3.  **Delegate Execution:** It delegates the core, resource-intensive logic to the Universe service, which manages the world lifecycle.
4.  **Handle Asynchronous Feedback:** It correctly handles the asynchronous nature of world loading by attaching callbacks to the returned CompletableFuture, ensuring the command sender receives feedback upon success or failure without blocking the main server thread.

This separation of concerns is critical for server stability. The command provides a safe, validated interface to a powerful backend system.

## Lifecycle & Ownership
-   **Creation:** A single instance of WorldLoadCommand is instantiated by the server's CommandSystem during the bootstrap phase. The system scans for classes extending CommandBase and registers them for runtime dispatch.
-   **Scope:** The WorldLoadCommand object is a long-lived singleton that persists for the entire server session. It acts as a stateless template for handling all `/load` command invocations.
-   **Destruction:** The instance is dereferenced and eligible for garbage collection only when the server shuts down and the CommandSystem is dismantled.

## Internal State & Concurrency
-   **State:** The class holds a single field, nameArg, which defines the command's argument structure. This state is configured in the constructor and is **immutable** for the object's lifetime. The class is otherwise stateless; all contextual data for an execution is provided via the CommandContext parameter.

-   **Thread Safety:** This class is thread-safe. The executeSync method is invoked by the server's main command processing thread. While it initiates a long-running asynchronous operation on a background thread pool via Universe.loadWorld, the WorldLoadCommand itself does not manage threads or shared mutable state. Its callbacks (thenRun, exceptionally) are safely executed by the Java CompletableFuture framework upon task completion.

    **Warning:** The Universe service, which this class interacts with, is expected to be a thread-safe singleton. Any concurrency issues would originate there, not within this command class.

## API Surface
The primary contract is fulfilled by overriding the executeSync method from its parent, CommandBase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(1) | Validates and initiates an asynchronous world loading operation. The complexity refers only to the invocation itself, not the subsequent background task which is highly complex. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. The system invokes it based on user input. The following example demonstrates how one might programmatically dispatch the command, which is the intended interaction model.

```java
// Example of dispatching the command via the central command system
CommandSystem commandSystem = server.getService(CommandSystem.class);
CommandSender console = server.getConsoleSender();

// The system will find the WorldLoadCommand and execute it
commandSystem.dispatch(console, "load my_persistent_world");
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new WorldLoadCommand()`. The command system manages the lifecycle of command objects. A manually created instance will not be registered and is non-functional.
-   **Direct Invocation:** Avoid calling `executeSync` directly. Doing so bypasses the command system's argument parsing, permission checks, and dispatch logic, which can lead to an inconsistent or insecure server state.
-   **Blocking on the Future:** The underlying `Universe.loadWorld` method is asynchronous for a reason: world loading is slow. Never attempt to block the server's main thread waiting for the load to complete. This command correctly uses non-blocking callbacks, and any similar implementation should follow this pattern.

## Data Pipeline
WorldLoadCommand acts as a controller in a control-flow pipeline, not a data-transformation pipeline. It translates a user command into a system-level asynchronous task.

> Flow:
> User Input (`/load world_name`) -> Command Parser -> **WorldLoadCommand** -> Universe.loadWorld() -> Asynchronous World Loader -> Command Callbacks -> User Feedback Message

