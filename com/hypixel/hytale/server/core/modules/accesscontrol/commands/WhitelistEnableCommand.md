---
description: Architectural reference for WhitelistEnableCommand
---

# WhitelistEnableCommand

**Package:** com.hypixel.hytale.server.core.modules.accesscontrol.commands
**Type:** Transient

## Definition
```java
// Signature
public class WhitelistEnableCommand extends CommandBase {
```

## Architecture & Concepts
The WhitelistEnableCommand class is a specific implementation of the Command Pattern, designed to handle a user-initiated request to enable the server's whitelist. It acts as a thin, stateless bridge between the server's command processing system and the core access control logic encapsulated within the HytaleWhitelistProvider.

Architecturally, this class decouples the command invocation (the user typing "/whitelist enable") from the state management of the whitelist system. Its sole responsibility is to receive a command context, delegate the action to the appropriate provider, and report the outcome back to the user. It contains no business logic beyond this delegation, ensuring a clean separation of concerns between user input handling and core server functionality.

This command is registered under the "enable" subcommand of the primary "whitelist" command, with an alias of "on". The framework ensures this class is only invoked when that specific command string is parsed.

### Lifecycle & Ownership
- **Creation:** Instantiated once during the server's bootstrap sequence, typically by its parent module, AccessControlModule. The required HytaleWhitelistProvider dependency is injected via the constructor at this time. The newly created instance is then registered with the central CommandManager.
- **Scope:** The object's lifecycle is tied to the server session. It persists as a registered handler for the entire duration the server is running.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the server shuts down or the AccessControlModule is unloaded.

## Internal State & Concurrency
- **State:** This class is **entirely stateless**. It maintains a single final field, whitelistProvider, which is a reference to an external service. It does not cache data or possess any mutable instance variables. All state changes are performed on the injected provider.

- **Thread Safety:** The primary entry point, executeSync, is explicitly designed to be executed on the main server thread. The command system framework guarantees this execution context. Direct invocation from asynchronous tasks or other threads is unsupported and will lead to race conditions within the underlying HytaleWhitelistProvider and CommandContext. The class is thread-safe only when used as intended within the server's single-threaded command execution model.

## API Surface
The public contract is fulfilled by its constructor for dependency injection and the overridden executeSync method for command execution.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| WhitelistEnableCommand(provider) | constructor | O(1) | Constructs the command handler, injecting the required whitelist provider. |
| executeSync(context) | void | O(1) | Toggles the whitelist state to enabled via the provider. Sends a status message to the command issuer. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically invoked by the server's command handling framework in response to player or console input.

A server administrator triggers this command's logic by executing the command in the server console or in-game chat:
```
/whitelist enable
```
or
```
/whitelist on
```
The framework then routes this request to the executeSync method of the registered WhitelistEnableCommand instance.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not manually create an instance using `new WhitelistEnableCommand(provider)`. The server's module and dependency injection system is responsible for its creation and registration. Manual instantiation will result in a "dangling" command object that is not registered to handle any input.
- **Direct Invocation:** Never call the executeSync method directly. Bypassing the command framework will skip critical permission checks, context setup, and thread safety guarantees, potentially corrupting server state. All command logic must be initiated through the server's command dispatcher.

## Data Pipeline
The flow of data for a successful command execution follows a clear path from user input to state change and feedback.

> Flow:
> User Input (`/whitelist enable`) -> Command Parser -> Command Registry Dispatcher -> **WhitelistEnableCommand.executeSync()** -> HytaleWhitelistProvider.setEnabled(true) -> CommandContext.sendMessage() -> Network Message -> Client UI Feedback

