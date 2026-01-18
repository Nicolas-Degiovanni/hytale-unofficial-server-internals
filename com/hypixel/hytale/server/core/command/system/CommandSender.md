---
description: Architectural reference for the CommandSender interface
---

# CommandSender

**Package:** com.hypixel.hytale.server.core.command.system
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface CommandSender extends IMessageReceiver, PermissionHolder {
```

## Architecture & Concepts
The CommandSender interface is a foundational abstraction within the server's command processing system. It does not represent a concrete object, but rather defines a contract for any entity capable of initiating a command. This design decouples the command execution logic from the source of the command, allowing for a unified handling mechanism.

By extending both IMessageReceiver and PermissionHolder, the CommandSender contract enforces two critical roles:
1.  **Principal:** As a PermissionHolder, every CommandSender is an actor within the server's security model. The CommandSystem uses this contract to verify if the sender has the required privileges before executing a command.
2.  **Recipient:** As an IMessageReceiver, every CommandSender has a designated channel to receive feedback, such as command success messages, error details, or usage instructions.

Common implementations include Players, the server Console, and automated in-game entities like Command Blocks. This interface is the primary subject passed to command handlers during execution.

## Lifecycle & Ownership
As an interface, CommandSender itself has no lifecycle. Its lifecycle is entirely dictated by the concrete object that implements it.

-   **Creation:** CommandSender is never instantiated directly. Concrete implementations are created by their managing systems. For example, a Player object (which implements CommandSender) is created by the network session manager upon a successful client connection. The server Console (another implementation) is typically a singleton created during server bootstrap.
-   **Scope:** The lifetime of a CommandSender is tied to its underlying implementation. A Player-backed sender is scoped to a player's session and becomes invalid upon disconnection. The Console-backed sender persists for the entire lifetime of the server process.
-   **Destruction:** Destruction is managed by the owner of the concrete implementation. For a Player, this occurs during the logout sequence.

**WARNING:** Caching or storing references to a CommandSender is highly discouraged. The underlying entity, especially a player, can be destroyed at any time, leading to stale references and unpredictable behavior.

## Internal State & Concurrency
-   **State:** The interface itself is stateless. However, all concrete implementations are inherently stateful. A Player implementation, for instance, holds mutable state regarding its location, inventory, and connection status.
-   **Thread Safety:** Implementations are **not thread-safe** and are designed to be accessed only from the main server thread. Command execution is strictly serialized to prevent race conditions. Any attempt to interact with a CommandSender from an asynchronous task or worker thread without proper synchronization will lead to severe concurrency issues, data corruption, or server crashes.

## API Surface
The primary contract of CommandSender is to provide a stable identity and inherit capabilities from its parent interfaces.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDisplayName() | String | O(1) | Retrieves the user-friendly name for this sender, e.g., a player's username. |
| getUuid() | UUID | O(1) | Retrieves the unique, persistent identifier for the sender. Essential for data storage and permission lookups. Returns a nil UUID for senders without a persistent identity, like the console. |

This interface also implicitly exposes the full API surface of IMessageReceiver (e.g., sendMessage) and PermissionHolder (e.g., hasPermission).

## Integration Patterns

### Standard Usage
The most common pattern is receiving the CommandSender as an argument in a command handler. The handler uses it to check permissions and provide feedback.

```java
// Inside a command handler method
public void execute(CommandSender sender, Arguments args) {
    if (!sender.hasPermission("myplugin.mycommand")) {
        sender.sendMessage("You do not have permission to use this command.");
        return;
    }

    // Command logic here...
    sender.sendMessage("Command executed successfully by " + sender.getDisplayName());
}
```

### Anti-Patterns (Do NOT do this)
-   **Unsafe Casting:** Never assume the CommandSender is a Player. The sender could be the console or another entity. Always perform an `instanceof` check before casting.
    ```java
    // BAD: Will crash if the sender is the console
    Player p = (Player) sender;
    p.teleport(...);

    // GOOD: Safely handles different sender types
    if (sender instanceof Player) {
        Player p = (Player) sender;
        p.teleport(...);
    } else {
        sender.sendMessage("This command can only be run by a player.");
    }
    ```
-   **Long-Lived References:** Do not store a CommandSender instance in a static field or long-lived cache. If the sender is a player who logs out, the reference becomes stale and will cause NullPointerExceptions or use-after-free errors.

## Data Pipeline
CommandSender is a key component in the server's command processing pipeline, acting as the context object for the execution phase.

> Flow:
> Client Input (e.g., Chat) -> Network Layer -> Command Dispatcher -> **CommandSender** (as argument) -> Command Handler -> Permission Check (via PermissionHolder) -> Feedback (via IMessageReceiver)

