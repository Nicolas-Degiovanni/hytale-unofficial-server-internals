---
description: Architectural reference for WhitelistListCommand
---

# WhitelistListCommand

**Package:** com.hypixel.hytale.server.core.modules.accesscontrol.commands
**Type:** Transient Command Object

## Definition
```java
// Signature
public class WhitelistListCommand extends CommandBase {
```

## Architecture & Concepts
The WhitelistListCommand class is a component of the server's Command System, implementing the Command design pattern. It encapsulates the specific logic required to execute the `list` subcommand for the server's whitelist functionality.

This class acts as a terminal node in the command processing chain. It serves as a clean, decoupled bridge between user-facing input (a typed command) and the server's core access control logic, which is abstracted away by the HytaleWhitelistProvider. Its sole responsibility is to query the provider for the current whitelist and format an appropriate response to the command's sender.

By extending CommandBase, it inherits the necessary boilerplate for command registration and execution, allowing it to focus exclusively on its specific task.

### Lifecycle & Ownership
- **Creation:** An instance of WhitelistListCommand is created during the server's module initialization phase. The Access Control module, or a similar bootstrap component, is responsible for instantiating it and injecting its required HytaleWhitelistProvider dependency.
- **Scope:** The object's lifecycle is tied to the command registry. It is created once upon server startup and persists for the entire server session, available to be executed whenever the corresponding command is invoked.
- **Destruction:** The instance is eligible for garbage collection upon server shutdown or when the command is explicitly deregistered from the command system.

## Internal State & Concurrency
- **State:** WhitelistListCommand is effectively stateless. It holds a single immutable reference to the HytaleWhitelistProvider, which is set at construction time. All operations are delegated to this provider, and the class itself does not cache or mutate any data between executions.
- **Thread Safety:** This class is **not thread-safe** and is designed to be operated exclusively by the main server thread. The `executeSync` method name is a strong contract indicating synchronous execution within the server's primary update loop. The Command System is responsible for ensuring this method is never invoked concurrently. Any attempt to call it from an external thread will lead to race conditions and undefined behavior.

## API Surface
The public contract is primarily defined by its constructor for dependency injection and the overridden `executeSync` method for execution by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| WhitelistListCommand(provider) | constructor | O(1) | Constructs the command object. Requires a non-null HytaleWhitelistProvider. |
| executeSync(context) | void | O(N) | Executes the command logic. Fetches N entries from the provider and sends a response to the user via the CommandContext. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is designed to be instantiated and registered with the server's command system, which then manages its execution lifecycle.

```java
// Example of registration within a module's initialization logic
HytaleWhitelistProvider provider = serviceRegistry.get(HytaleWhitelistProvider.class);
CommandRegistry commandRegistry = serviceRegistry.get(CommandRegistry.class);

// Assume a parent "whitelist" command exists
CommandNode parentCommand = commandRegistry.getNode("whitelist");
parentCommand.register(new WhitelistListCommand(provider));
```

### Anti-Patterns (Do NOT do this)
- **Direct Invocation:** Never call the `executeSync` method directly. The Command System must provide the `CommandContext`, which contains critical information about the command sender and the execution environment. Bypassing the system will result in a failure to deliver a response and breaks the command flow.
- **Stateful Implementation:** Do not modify this class to cache the whitelist. The HytaleWhitelistProvider is the single source of truth, and caching here would introduce stale data and synchronization problems.

## Data Pipeline
The flow of data for a typical execution of this command is unidirectional, originating from user input and terminating in a message sent back to that user.

> Flow:
> User Command (`/whitelist list`) -> Server Network Listener -> Command Parser -> Command Dispatcher -> **WhitelistListCommand.executeSync()** -> HytaleWhitelistProvider.getList() -> Formatted Message -> CommandContext.sendMessage() -> Server Network Egress -> User's Console

