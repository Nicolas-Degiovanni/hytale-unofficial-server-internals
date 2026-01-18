---
description: Architectural reference for KickCommand
---

# KickCommand

**Package:** com.hypixel.hytale.server.core.command.commands.server
**Type:** Handler

## Definition
```java
// Signature
public class KickCommand extends CommandBase {
```

## Architecture & Concepts
The KickCommand class is a concrete implementation within the server's Command System framework. It encapsulates the logic required to execute the administrative `/kick` command, which forcibly disconnects a specified player from the server.

This class does not handle input parsing or command routing. Instead, it declares its required arguments—in this case, a target player—using the framework's argument definition system. Its sole responsibility is to act upon a valid, parsed command invocation delivered by the command processor.

Architecturally, it serves as a direct bridge between a high-level administrative intent (kicking a player) and a low-level network action (terminating a client's session via its PacketHandler).

### Lifecycle & Ownership
-   **Creation:** A single instance of KickCommand is instantiated by the server's central CommandRegistry during the server bootstrap sequence. The registry scans for all classes extending CommandBase and creates long-lived instances.
-   **Scope:** Application-scoped. The KickCommand object persists for the entire lifetime of the server process. It is not created on a per-request basis.
-   **Destruction:** The object is de-referenced and eligible for garbage collection only upon server shutdown when the CommandRegistry is cleared.

## Internal State & Concurrency
-   **State:** KickCommand is effectively stateless and immutable after its initial construction. Its fields, which define argument requirements and response messages, are initialized once and are not modified during command execution. All state relevant to a specific invocation is passed via the CommandContext parameter.
-   **Thread Safety:** The class itself is inherently thread-safe due to its stateless design. However, the `executeSync` method name is a strong convention indicating that the command framework guarantees its execution on the main server thread. Modifying this class to introduce asynchronous operations or shared mutable state would violate the core contract of the Command System and lead to severe concurrency issues.

## API Surface
The primary interaction surface is the `executeSync` method, which is invoked by the command system, not by developers directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(context) | void | O(1) | Executes the kick logic. Retrieves the target PlayerRef from the context, accesses its network PacketHandler, and issues a disconnect. Sends a confirmation message to the command issuer. |

## Integration Patterns

### Standard Usage
This class is not used directly. It is invoked by the server's command processing system when a user with appropriate permissions executes the corresponding command.

```java
// This logic is handled internally by the server's command processor.
// A developer would never write this code.

// 1. User types "/kick Notch"
// 2. Server parses this into a command and its arguments.
// 3. The CommandRegistry dispatches to the KickCommand instance.
CommandContext context = createFromUserInput("/kick Notch");
KickCommand kickCmd = commandRegistry.get("kick");
kickCmd.executeSync(context); // Framework invokes the handler
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new KickCommand()`. An instance created this way is not registered with the command system and will never be executed. It is a dead object.
-   **Manual Invocation:** Do not call `executeSync` directly. Bypassing the command framework skips critical operations like permission checks, argument parsing, and context population, which will result in runtime exceptions or undefined behavior.

## Data Pipeline
The flow of data for a kick operation begins with user input and terminates with a network-level disconnection.

> Flow:
> User Input (`/kick <username>`) -> Network Ingress -> Command Parser -> **KickCommand.executeSync** -> PlayerRef.getPacketHandler().disconnect() -> Network Egress (Disconnect Packet) -> Client Disconnection

