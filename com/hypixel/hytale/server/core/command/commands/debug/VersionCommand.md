---
description: Architectural reference for VersionCommand
---

# VersionCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug
**Type:** Transient

## Definition
```java
// Signature
public class VersionCommand extends CommandBase {
```

## Architecture & Concepts
The VersionCommand class is a concrete implementation of the Command Pattern, designed to be discovered and managed by the server's central command system. Its sole responsibility is to retrieve static build and version information from the application's JAR manifest and report it back to the command's issuer.

This class acts as a terminal endpoint in the command processing pipeline. It encapsulates a single, well-defined server action—displaying the version—decoupling the command invocation (e.g., a user typing in the console) from the underlying logic that fetches the data. It relies on the ManifestUtil utility to abstract the details of reading from the manifest file, ensuring this command's logic remains simple and focused.

## Lifecycle & Ownership
- **Creation:** An instance of VersionCommand is created by the command registration service during server bootstrap. The system scans for all subclasses of CommandBase and instantiates them to build a registry of available commands.
- **Scope:** The singleton instance of this command persists for the entire server session, held within the central command registry. It does not maintain state between executions.
- **Destruction:** The object is dereferenced and eligible for garbage collection during server shutdown when the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is stateless. The two Message fields are static, final, and immutable templates for localization. All data required for execution is either static (from ManifestUtil) or provided transiently via the CommandContext argument.
- **Thread Safety:** The class is inherently thread-safe. The executeSync method is designed to be called synchronously by the server's main command processing thread. It performs read-only operations on globally accessible, immutable data. The CommandContext object is unique per invocation, preventing any risk of cross-thread state corruption.

## API Surface
The public contract is defined by its superclass, CommandBase. The primary interaction point is the execution method invoked by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| VersionCommand() | constructor | O(1) | Initializes the command with its name ("version") and description. |
| executeSync(CommandContext) | protected void | O(1) | Executes the command's logic. Reads version data and sends a formatted response message via the provided context. |

## Integration Patterns

### Standard Usage
This command is not intended to be invoked directly from code. It is automatically discovered and executed by the server's command handler in response to user input. A server administrator or a player with sufficient permissions would trigger it.

*Example User Interaction:*
1. User connects to the server.
2. User types `/version` into the chat or console.
3. The server's command system parses the input, identifies the "version" command, and finds the registered VersionCommand instance.
4. The system invokes `executeSync`, passing a CommandContext representing the user.
5. The command sends the version information back to the user's chat window.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of this class manually. The command system handles its lifecycle. Instantiating it yourself will result in an object that is not registered and therefore cannot be invoked by name.
- **Manual Execution:** Avoid calling the executeSync method directly. Doing so bypasses the command system's essential middleware, including permission checks, cooldowns, and logging. Always delegate command execution to the central command dispatcher.

## Data Pipeline
The flow of data for a typical version check is linear and synchronous, originating from user input and terminating with a network message.

> Flow:
> User Input (`/version`) -> Network Packet -> Server Command Parser -> Command Dispatcher -> **VersionCommand.executeSync()** -> ManifestUtil -> Message Formatter -> Network Layer -> Client Chat UI

