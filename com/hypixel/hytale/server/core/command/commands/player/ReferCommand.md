---
description: Architectural reference for ReferCommand
---

# ReferCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player
**Type:** Transient

## Definition
```java
// Signature
public class ReferCommand extends AbstractTargetPlayerCommand {
```

## Architecture & Concepts
The ReferCommand class is a concrete implementation within the server's Command System framework. It encapsulates the logic for the administrative command `/refer`, which initiates a client-side transfer of a player to a different game server.

This class adheres to the Command Pattern. Its primary responsibility is not to perform the network transfer itself, but to act as a secure bridge between user input and the core player session management system. It achieves this by:
1.  Defining the command's syntax, including required arguments like *host* and *port*.
2.  Integrating with the `CommandContext` to parse and validate user-provided arguments.
3.  Enforcing strict, context-aware permission checks using the `HytalePermissions` system.
4.  Delegating the final, sensitive operation of player referral to the `PlayerRef` entity component.

By extending `AbstractTargetPlayerCommand`, it leverages a shared infrastructure for resolving player names to entity references, reducing boilerplate code and centralizing player lookup logic.

## Lifecycle & Ownership
- **Creation:** A single prototype instance of ReferCommand is instantiated by the `CommandRegistry` during server bootstrap. The registry scans for classes annotated as commands and creates long-lived instances to handle all future invocations.
- **Scope:** The prototype instance is a singleton within the context of the `CommandRegistry` and persists for the entire server runtime. The object itself is stateless; all contextual data for a specific execution is passed into the `execute` method.
- **Destruction:** The instance is dereferenced and eligible for garbage collection only when the server shuts down and the `CommandRegistry` is cleared.

## Internal State & Concurrency
- **State:** ReferCommand is effectively stateless and immutable after construction. Its fields, `hostArg` and `portArg`, are final and define the command's argument structure, not its runtime state. All execution-specific data is scoped to the `execute` method's parameters.
- **Thread Safety:** This class is thread-safe by virtue of its stateless design. However, the `execute` method is designed to be invoked exclusively by the server's main command processing thread. Concurrent, external calls to `execute` would bypass the command system's serialization and context management, leading to unpredictable behavior. The framework guarantees safe, sequential execution.

## API Surface
The primary contract is the `execute` method, inherited and implemented from the command system's base classes.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | protected void | O(1) | Parses arguments, validates permissions, and triggers the player referral process. Complexity is constant time, but it invokes `PlayerRef.referToServer`, which involves network I/O and has variable real-world latency. |

## Integration Patterns

### Standard Usage
This command is intended to be invoked by a privileged user through the in-game chat or server console. The command system handles parsing, object resolution, and invocation.

```java
// This logic is handled internally by the Command System.
// A user would simply type the command in chat.

// 1. User types: /refer PlayerName some.server.com 25565
// 2. Command system parses the input.
// 3. It finds the registered ReferCommand instance.
// 4. It invokes execute() with a fully populated CommandContext.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ReferCommand()`. The command system relies on a single, registered instance for routing. A manually created instance will not be executable via chat or console.
- **Direct Invocation:** Avoid calling the `execute` method directly. Doing so bypasses the argument parsing and context setup provided by the `AbstractTargetPlayerCommand` superclass, which will result in runtime exceptions and incorrect state.
- **Permission Bypass:** The permission checks inside `execute` are critical for server security. Any attempt to modify or circumvent these checks can create a severe security vulnerability, allowing unauthorized players to force-transfer other users.

## Data Pipeline
The flow of data for a successful referral operation begins with user input and terminates with a network instruction sent to the player's client.

> Flow:
> Player Chat Input -> Server Network Listener -> Command Parser -> **ReferCommand.execute()** -> PlayerRef.referToServer() -> Server Network Emitter -> Referral Packet to Client

