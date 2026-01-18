---
description: Architectural reference for PlayLanCommand
---

# PlayLanCommand

**Package:** com.hypixel.hytale.server.core.modules.singleplayer.commands
**Type:** Transient

## Definition
```java
// Signature
public class PlayLanCommand extends PlayCommandBase {
```

## Architecture & Concepts
The PlayLanCommand class is a concrete implementation of the **Command Pattern**, designed to encapsulate the action of opening a single-player world to the Local Area Network (LAN). It exists as a specialized variant of the more generic PlayCommandBase, from which it inherits its core execution logic.

This class does not contain any operational logic itself. Its primary architectural role is to provide a specific configuration to its parent class during construction. By passing the enum value Access.LAN to the PlayCommandBase constructor, it instructs the underlying system to initialize a game session with network visibility and authentication rules appropriate for a LAN environment.

This command is intrinsically linked to the SingleplayerModule, which manages the lifecycle of a single-player server instance. It is only intended to be registered and executed within the context of that module.

### Lifecycle & Ownership
- **Creation:** Instantiated once by the SingleplayerModule during its own initialization phase. The module is responsible for creating and subsequently registering the command with a central command processing service.
- **Scope:** The PlayLanCommand instance persists for the entire lifetime of the SingleplayerModule. Its lifecycle is directly bound to the server operating in a single-player mode.
- **Destruction:** The object is marked for garbage collection when the SingleplayerModule is shut down or the server transitions to a different mode, at which point it is deregistered from the command system.

## Internal State & Concurrency
- **State:** This class is stateless. It contains no member fields and its behavior is defined entirely by the arguments passed to its parent constructor. After instantiation, its configuration is effectively **immutable**.
- **Thread Safety:** The object itself is thread-safe due to its immutability. However, the action it triggers—modifying the server state to open to LAN—is a high-impact operation. The command execution framework **must** ensure that this command is only ever invoked on the main server thread to prevent catastrophic race conditions and state corruption.

## API Surface
The public contract of PlayLanCommand is minimal, consisting only of its constructor. All operational methods, such as an execute or process method, are inherited from its parent, PlayCommandBase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PlayLanCommand(singleplayerModule) | constructor | O(1) | Constructs and configures the command for LAN play. Requires a valid SingleplayerModule context. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is designed to be registered with a command handler as part of a module's startup routine. The system then invokes the command in response to user input.

```java
// Correct registration within the SingleplayerModule
// This code is conceptual and demonstrates the pattern.

CommandRegistry commandRegistry = serverContext.getCommandRegistry();
commandRegistry.register(new PlayLanCommand(this));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation and Execution:** Do not create an instance of this class and attempt to call its methods directly. This bypasses the server's command processing pipeline, including essential permission checks and argument parsing.
- **Incorrect Context:** Instantiating this command without a valid, active SingleplayerModule will lead to runtime exceptions or undefined behavior. It is fundamentally coupled to the single-player game loop.

## Data Pipeline
PlayLanCommand acts as an endpoint in the server's command processing flow. It translates a user-initiated text command into a system-level action.

> Flow:
> User Input (`/play lan`) -> Server Command Parser -> Command Registry Lookup -> **PlayLanCommand.execute()** -> SingleplayerModule.openToLan() -> Network Subsystem Configuration

