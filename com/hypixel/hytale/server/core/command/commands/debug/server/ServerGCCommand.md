---
description: Architectural reference for ServerGCCommand
---

# ServerGCCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug.server
**Type:** Handler

## Definition
```java
// Signature
public class ServerGCCommand extends CommandBase {
```

## Architecture & Concepts
The ServerGCCommand class is a concrete implementation of the Command Pattern, designed to integrate with the server's central command processing system. Its sole responsibility is to expose a low-level Java Virtual Machine (JVM) operation—forced garbage collection—to a privileged user, such as a server administrator.

This class acts as a bridge between the server's command interpreter and the underlying Java runtime. It is not part of the core game loop or gameplay logic; instead, it serves as a diagnostic and administrative tool. The CommandBase superclass provides the necessary boilerplate for command registration and lifecycle management, allowing ServerGCCommand to focus exclusively on its execution logic. By inheriting from CommandBase, it adheres to the contract required by the command system, which discovers and invokes it at the appropriate time.

### Lifecycle & Ownership
- **Creation:** An instance of ServerGCCommand is created during the server's bootstrap sequence. A central CommandRegistry or a similar service manager is responsible for scanning, instantiating, and registering all command handlers, including this one.
- **Scope:** The object's lifecycle is tied to the CommandRegistry. It persists for the entire duration of the server session, from startup to shutdown.
- **Destruction:** The object is dereferenced and becomes eligible for garbage collection when the server shuts down and the CommandRegistry is cleared. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** ServerGCCommand is **stateless**. It contains no member fields and does not cache data between invocations. Its behavior is entirely dependent on the arguments passed via the CommandContext and the current memory state of the JVM.
- **Thread Safety:** This class is not designed for concurrent invocation. The `executeSync` method name strongly implies that the command system guarantees its execution on a specific, synchronized server thread (e.g., the main server tick thread). The underlying call to System.gc is a system-wide, blocking operation managed by the JVM. Developers must not invoke this class's methods from asynchronous threads.

## API Surface
The public contract is defined by its constructor for instantiation by the framework and the protected `executeSync` method, which fulfills the CommandBase contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ServerGCCommand() | constructor | O(1) | Initializes the command with its name ("gc") and description key. |
| executeSync(context) | protected void | Variable | Triggers a full, system-wide JVM garbage collection. The performance cost is unpredictable and can cause significant server pauses. Sends a confirmation message to the user via the CommandContext. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly by developers in game logic. The system discovers and registers it automatically. A server administrator invokes it by typing the command into the server console or in-game chat.

> **Administrator Interaction Flow**
> 1. An administrator connects to the server and types `/gc` in the console.
> 2. The server's command parser identifies "gc" as a valid command.
> 3. The CommandSystem retrieves the registered ServerGCCommand instance.
> 4. The system invokes the `executeSync` method on the main server thread, passing the administrator's context.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ServerGCCommand()` in your code. The object is useless unless registered with the server's command system, which is handled automatically at startup.
- **Direct Invocation:** Never call the `executeSync` method directly. Doing so bypasses the command system's essential cross-cutting concerns, such as permission checks, thread safety guarantees, and logging.
- **Frequent Execution:** This command is a blunt instrument for diagnostics. Triggering garbage collection frequently can severely degrade server performance by inducing "stop-the-world" pauses. It should only be used to investigate specific memory-related issues.

## Data Pipeline
The data flow for this command is initiated by user input and terminates with a feedback message, with the ServerGCCommand acting as the central handler for a JVM-level side effect.

> Flow:
> User Input (`/gc`) -> Network Layer -> Command Parser -> **CommandSystem Dispatcher** -> ServerGCCommand.executeSync() -> JVM Runtime -> CommandContext -> Message Formatter -> Network Layer -> Client Chat UI

