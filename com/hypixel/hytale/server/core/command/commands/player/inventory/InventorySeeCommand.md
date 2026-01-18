---
description: Architectural reference for InventorySeeCommand
---

# InventorySeeCommand

**Package:** com.hypixel.hytale.server.core.command.commands.player.inventory
**Type:** Singleton

## Definition
```java
// Signature
public class InventorySeeCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The InventorySeeCommand class is a server-side command handler that allows a privileged user, typically an administrator, to view the inventory of another player in real-time. It serves as a critical orchestration component, bridging several core engine systems: the Command System, the Entity Component System (ECS), the World Threading Model, and the Player UI System.

Architecturally, its primary responsibility is to safely access entity data from a different world context and present it to the command's sender. It achieves this by:
1.  Parsing a target player argument from the command context.
2.  Locating the target player's entity reference (PlayerRef) and its associated World.
3.  **Scheduling a task on the target player's World thread.** This is the most critical architectural pattern employed here, ensuring that all interactions with the target's components are thread-safe.
4.  Applying a security-driven Decorator pattern. If the command sender lacks modification permissions, the target's inventory container is wrapped in a read-only DelegateItemContainer, preventing any changes.
5.  Instructing the sender's PageManager to open a new UI window (Page.Bench) populated with the target's inventory data.

This command exemplifies the separation of concerns within the server architecture: the command itself is a lightweight dispatcher, while the heavy lifting of data access and UI presentation is delegated to the World and Player component systems respectively.

## Lifecycle & Ownership
-   **Creation:** A single instance of InventorySeeCommand is instantiated by the server's command registration system during the server bootstrap sequence. It is registered under the name "see".
-   **Scope:** The object instance is a singleton that persists for the entire server session. It does not hold state related to any specific execution, making it reusable.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
-   **State:** The class is effectively stateless after construction. Its only instance field, targetPlayerArg, is an argument definition object initialized in the constructor and is not mutated during command execution. All state required for execution is provided via the CommandContext and other parameters passed to the execute method.

-   **Thread Safety:** The class itself is thread-safe due to its stateless nature. However, its *operation* is highly sensitive to threading. The execute method is invoked on the command executor thread. It correctly handles cross-thread entity access by using `targetWorld.execute`.

    > **Warning:** Any modification to this class that attempts to access the target player's components directly from the initial calling thread will introduce severe race conditions and data corruption. All interactions with a player's inventory or other components **must** be marshaled to that player's owning World thread.

## API Surface
The public contract is defined by its role as a command. Direct programmatic invocation is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | protected void | O(1) | Overrides AbstractPlayerCommand. Orchestrates the entire inventory viewing process, from argument parsing to UI packet generation. Throws no checked exceptions but sends error messages to the user on failure. |

## Integration Patterns

### Standard Usage
This command is not intended for direct invocation via code. It is designed to be executed by a player through the in-game chat console.

```
// Player enters the following command in chat
/invsee OtherPlayerName
```
Upon execution, the server will open a user interface for the command sender, displaying the live inventory of the player named OtherPlayerName.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new InventorySeeCommand()`. The command system manages the lifecycle of this object.
-   **Direct Method Invocation:** Do not call the `execute` method directly. This bypasses critical infrastructure such as argument parsing, permission validation, and context setup provided by the command system.
-   **Cross-Thread Component Access:** Do not attempt to retrieve the target player's inventory or other components without scheduling the logic on the target's World thread via `world.execute`. This will lead to server instability and crashes.

## Data Pipeline
The flow of data and control for this command crosses multiple threads and systems, from user input to client-side rendering.

> Flow:
> Player Chat Input -> Server Command Parser -> **InventorySeeCommand.execute()** -> World Thread Scheduler -> [On Target Player's World Thread] Fetch Player Component & Inventory -> [Permission Check] Wrap Inventory in Read-Only Delegate (if needed) -> [On Sender's Network Thread] Player.PageManager.setPageWithWindows() -> Network Packet (S2C) -> Client UI System -> Render Inventory Window

