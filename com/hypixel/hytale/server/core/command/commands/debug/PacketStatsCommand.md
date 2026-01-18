---
description: Architectural reference for PacketStatsCommand
---

# PacketStatsCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug
**Type:** Transient

## Definition
```java
// Signature
public class PacketStatsCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The PacketStatsCommand is a server-side diagnostic tool that provides a user-facing interface to the low-level network monitoring subsystem. It operates within the server's Command System, acting as a concrete implementation of the Command Pattern.

Its primary function is to retrieve, format, and display detailed statistics for specific packet types associated with a player's network session. This class serves as a crucial bridge between an administrator's request and the underlying PacketStatsRecorder, which is deeply embedded within a player's PacketHandler.

By extending AbstractTargetPlayerCommand, this class delegates the complex logic of parsing and resolving player targets, allowing it to focus solely on the business logic of retrieving and presenting packet data. This inheritance is a key aspect of the command system's design, promoting high cohesion and low coupling for commands that operate on player entities.

**WARNING:** This is a debug command and is not intended for use in production environments without appropriate performance considerations. Frequent execution may have minor overhead.

## Lifecycle & Ownership
- **Creation:** A single instance of PacketStatsCommand is instantiated by the server's command registration service during the server bootstrap sequence. It is then registered under the name "packetstats".
- **Scope:** The command object itself is a singleton that persists for the entire server lifetime. However, its execution is ephemeral. Each invocation of the command is a distinct event, driven by a new CommandContext object that is discarded after the command completes.
- **Destruction:** The singleton instance is de-referenced and eligible for garbage collection only when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless. Its only instance field, packetArg, is an immutable definition of a required command argument, configured during construction. All operational state, such as the target player and command arguments, is passed into the execute method via the CommandContext.
- **Thread Safety:** The PacketStatsCommand object is inherently thread-safe due to its stateless design. However, it reads data from the PacketStatsRecorder, which is mutated concurrently by the server's network I/O threads. The PacketStatsRecorder is responsible for its own internal synchronization. This command acts as a safe, read-only client to that concurrent data structure.

## API Surface
The public API is minimal, designed to be invoked exclusively by the server's command processing system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PacketStatsCommand() | constructor | O(1) | Initializes the command and defines its required arguments. For internal use by the command system. |
| execute(...) | void | O(N) | Executes the command logic. Retrieves the PacketStatsRecorder for the target player and queries it for the specified packet. N is the number of packet types with recorded data, with a maximum of 512. |

## Integration Patterns

### Standard Usage
An administrator or developer invokes this command through the server console or in-game chat. The command system parses the input, identifies the registered PacketStatsCommand instance, and invokes its execute method with a fully populated context.

```java
// This code is conceptual. Direct invocation is an anti-pattern.
// The command is typically run via the server console or chat:
// > /packetstats PlayerName PacketEntityUpdate

// The system internally resolves this to an invocation similar to:
CommandContext context = createFromInput("/packetstats PlayerName PacketEntityUpdate");
PacketStatsCommand command = commandRegistry.get("packetstats");
command.execute(context, ...);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PacketStatsCommand()`. The server's command registry is the sole owner of the command's lifecycle. Manually creating an instance will result in a non-functional object that is not registered to handle any commands.
- **Manual Invocation:** Avoid calling the `execute` method directly. Doing so bypasses critical infrastructure provided by the command system, including argument parsing, validation, permission checks, and sender resolution.
- **State Assumption:** Never assume that packet statistics are being recorded. The code correctly checks if the PacketStatsRecorder is null, as this feature can be disabled. Any system integrating with this data must perform the same null check.

## Data Pipeline
PacketStatsCommand functions as a read-only endpoint in a diagnostic data flow. It translates a high-level user request into a formatted, human-readable report from a low-level, high-throughput data source.

> Flow:
> User Input (Chat/Console) -> Command System Parser -> **PacketStatsCommand.execute()** -> PlayerRef -> PacketHandler.getPacketStatsRecorder() -> Data Aggregation & Formatting -> Message Bus -> User (Chat/Console)

