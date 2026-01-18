---
description: Architectural reference for BrushConfigLoadCommand
---

# BrushConfigLoadCommand

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.commands
**Type:** Transient Handler

## Definition
```java
// Signature
public class BrushConfigLoadCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The BrushConfigLoadCommand serves as the primary command-line interface for a player to manage their active Scripted Brush. It is a critical entry point into the Builder Tools system, acting as a controller that translates player text input into direct state changes on the player's entity components.

This command is architecturally significant for its dual-mode operation:

1.  **UI Invocation Mode (No Arguments):** When executed without arguments, the command does not directly manipulate brush data. Instead, it functions as a bridge to the UI system, instructing the player's client to open the ScriptedBrushPage. This delegates the selection process to a graphical interface, providing a user-friendly discovery and selection mechanism.

2.  **Direct Load Mode (With Arguments):** When a brush asset name is provided, the command bypasses the UI entirely. It directly resolves the specified ScriptedBrushAsset and orchestrates the loading of its configuration into the player's active toolset. This provides a power-user workflow for rapidly switching between known brushes.

In both modes, the command's primary responsibility is to safely mutate the state held within the player's **PrototypePlayerBuilderToolSettings** object. It performs crucial safety checks to prevent state corruption, most notably by verifying that no brush operation is currently in progress.

## Lifecycle & Ownership
-   **Creation:** A single instance of BrushConfigLoadCommand is instantiated by the server's core Command System during the server bootstrap and command registration phase.
-   **Scope:** The command object is a long-lived singleton that persists for the entire server session. However, its execution context is transient; the execute method is invoked for a brief moment in response to a specific player command and does not persist.
-   **Destruction:** The instance is garbage collected when the server shuts down and the Command System is dismantled.

## Internal State & Concurrency
-   **State:** This class is fundamentally stateless. It does not retain any data between invocations. All state it reads from and writes to, such as the player's current brush configuration, is stored externally in ECS components, primarily PrototypePlayerBuilderToolSettings. The nested LoadByNameCommand class holds a reference to its argument definition, but this is configured at creation and is immutable thereafter.

-   **Thread Safety:** This command is **not thread-safe** and must be executed exclusively on the main server thread. Its operations involve direct access to the Entity-Component-System (ECS) store, which is not designed for concurrent access. The system relies on the Command System's execution model to guarantee single-threaded invocation.

    **WARNING:** The `brushConfig.isCurrentlyExecuting()` check is a critical, non-atomic guard against logical race conditions. It prevents a player from loading a new brush while another is still performing a world modification, which would lead to undefined behavior and potential world data corruption.

## API Surface
The public API is not intended for direct developer invocation but is instead consumed by the server's Command System. The logical operations exposed to players are:

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| brush load | void | O(1) | Invokes the UI system. Opens the ScriptedBrushPage for the executing player. Fails if a brush is currently executing. |
| brush load <brushName> | void | O(N) | Loads a ScriptedBrushAsset by its asset ID. Complexity depends on asset loading and script compilation. Fails if a brush is currently executing. |

## Integration Patterns

### Standard Usage
A developer or player interacts with this system via the in-game chat or server console. The Command System routes the input to this handler.

```java
// This code is conceptual and represents how the Command System
// would dispatch an invocation. Do not call this directly.

// Simulating a player typing "/brush load my_terrain_brush"
CommandContext context = createMockContextForPlayer("my_terrain_brush");
PlayerRef player = context.getSource().getPlayer();
Store<EntityStore> store = server.getWorld().getStore();
Ref<EntityStore> ref = player.getRef();

// The Command System finds the registered BrushConfigLoadCommand instance
// and invokes its logic.
registeredCommand.process(context, store, ref, player, world);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new BrushConfigLoadCommand()`. The command must be registered with the server's Command System to function. Manual instantiation provides no value and will not be executable.
-   **Bypassing Safety Checks:** Logic that replicates this command's behavior must also replicate the `isCurrentlyExecuting()` check. Failure to do so risks corrupting the player's tool state and causing unpredictable world edits.
-   **Off-Thread Execution:** Attempting to invoke this command's logic from a background thread will result in severe concurrency exceptions when accessing the ECS store and player components.

## Data Pipeline
The command acts as a controller in a user-driven data flow. It does not process a stream of data but rather initiates a state change based on a single event.

> **UI Flow:**
> Player Input (`/brush load`) -> Command Parser -> **BrushConfigLoadCommand.execute()** -> Player.getPageManager() -> Client-Side UI (`ScriptedBrushPage`)

> **Direct Load Flow:**
> Player Input (`/brush load my_brush`) -> Command Parser -> **LoadByNameCommand.execute()** -> AssetArgumentType (Resolves Asset) -> BrushConfigCommandExecutor (Loads Asset) -> PrototypePlayerBuilderToolSettings (State Updated) -> PlayerRef.sendMessage() (Feedback)

