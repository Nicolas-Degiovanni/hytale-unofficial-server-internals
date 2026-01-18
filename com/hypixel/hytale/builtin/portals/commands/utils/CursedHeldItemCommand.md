---
description: Architectural reference for CursedHeldItemCommand
---

# CursedHeldItemCommand

**Package:** com.hypixel.hytale.builtin.portals.commands.utils
**Type:** Transient

## Definition
```java
// Signature
public class CursedHeldItemCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The CursedHeldItemCommand is a server-side component that implements the Command Pattern. It provides in-game administrative functionality for players to modify the state of an item they are holding. As a subclass of AbstractPlayerCommand, it is intrinsically tied to the server's command processing system and is designed to be executed within the context of a specific player entity.

Its core function is to access the player's inventory, retrieve the currently held ItemStack, and toggle a boolean "cursed" flag within its AdventureMetadata. This is a direct, synchronous mutation of game state. The command handles the logic for fetching the component data from the Entity-Component-System (ECS) storage, performing the modification, and writing the updated ItemStack back to the player's inventory. It also provides localized feedback messages to the executing player.

## Lifecycle & Ownership
-   **Creation:** An instance of CursedHeldItemCommand is created by the server's command registry during the bootstrap phase or when a module containing it is loaded. It is registered under the alias "cursethis".
-   **Scope:** The command object itself is a long-lived singleton for the duration of the server's runtime. It is stateless, and its methods are re-entrant with different contexts.
-   **Destruction:** The instance is destroyed and garbage collected when the server shuts down or the command registry is cleared.

## Internal State & Concurrency
-   **State:** This class is stateless. All data required for its operation, such as the player reference and world store, is passed as arguments to the execute method. The only internal field is a static, immutable Message constant for localization.
-   **Thread Safety:** This class is **not thread-safe** and must not be treated as such. The execute method performs direct, unsynchronized mutations on the game world state (EntityStore). The server's architecture guarantees that all command executions occur serially on the main game loop thread, preventing race conditions. Any attempt to invoke this method from an asynchronous task or a different thread will lead to world corruption.

## API Surface
The public API is defined by its parent, AbstractPlayerCommand. The primary entry point is the overridden execute method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(1) | Toggles the "cursed" metadata on the player's held item. Sends feedback messages to the player. |

## Integration Patterns

### Standard Usage
This command is not intended to be used directly via code. It is invoked by the game engine when a player with appropriate permissions types the command into the chat console.

> Player types `/cursethis` -> Server Command Dispatcher -> CursedHeldItemCommand.execute()

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new CursedHeldItemCommand()`. The command will not be registered with the server and will have no effect. The command system handles instantiation and registration.
-   **Manual Invocation:** Do not call the `execute` method directly. This bypasses the entire command processing pipeline, including permission checks, argument parsing, and context validation. This is a critical violation of the engine's design and can lead to unstable server state.

## Data Pipeline
The data flow for this command is initiated by a player and results in a direct mutation of an entity's component data.

> Flow:
> Player Chat Input -> Network Packet -> Server Command Parser -> **CursedHeldItemCommand.execute()** -> EntityStore Read (Player Inventory) -> ItemStack Metadata Mutation -> EntityStore Write (Player Inventory) -> Player Feedback Message

