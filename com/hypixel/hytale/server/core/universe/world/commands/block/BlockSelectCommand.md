---
description: Architectural reference for BlockSelectCommand
---

# BlockSelectCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.block
**Type:** Transient

## Definition
```java
// Signature
public class BlockSelectCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The BlockSelectCommand is a server-side administrative and developer tool that implements the Command design pattern. It integrates with the server's command processing system to provide a powerful mechanism for querying, filtering, and visualizing game blocks in-world.

Its primary function is to bridge the gap between the abstract asset data stored in the BlockTypeAssetMap and a tangible, spatial representation. Upon execution, it performs a complex query against all registered block types, applying filters for names (via regex), properties like flip type, and rotation variants. The resulting set of blocks is then algorithmically arranged in a grid and presented to the player using the SelectionManager.

This command is not intended for standard gameplay. It serves as a crucial utility for content creators, world builders, and engine developers to debug block assets, verify properties, and quickly build palettes for creative projects. It directly manipulates a player's selection provider, a temporary client-side construct, to render the results without permanently modifying the world state.

## Lifecycle & Ownership
- **Creation:** A single instance of BlockSelectCommand is created by the server's command registration system during the server bootstrap phase. The system scans for command implementations and instantiates them for its internal registry.
- **Scope:** The BlockSelectCommand object is a long-lived singleton within the command registry, persisting for the entire server session. However, the execution context and all variables within the execute method are transient and exist only for the duration of a single command invocation.
- **Destruction:** The instance is garbage collected when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** The class instance is effectively immutable after construction. Its fields are final references to argument definition objects (e.g., regexArg, paddingArg). These objects define the command's signature but do not hold state between executions. All state relevant to a specific invocation is passed as parameters to the execute method.
- **Thread Safety:** This class is thread-safe. The execute method is invoked by the server's main logic thread. The initial data processing stage leverages a parallel stream (`parallelStream()`) to filter and sort block assets. This is safe because it reads from the globally accessible and presumably immutable BlockTypeAssetMap. The final stage, which generates the in-world selection, is handed off to the SelectionProvider via a lambda. This pattern ensures that the actual world-interaction logic is executed within a context that is synchronized with the game loop, preventing race conditions.

## API Surface
The public API is defined by its role as a command handler. Direct invocation outside the command system is not supported.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, store, ref, playerRef, world) | void | O(N log N) | The command entry point. Complexity is dominated by the sorting of N block types. Filters assets, calculates a grid layout, and uses the SelectionManager to generate a client-side visual selection. Throws GeneralCommandException for invalid arguments. |

## Integration Patterns

### Standard Usage
A user with appropriate permissions invokes this command through the in-game chat console. The server's command dispatcher identifies the command, parses the arguments, and invokes the execute method on the registered BlockSelectCommand instance.

```
// Player chat input
/blockselect regex=.*Door.* sort=key padding=2
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new BlockSelectCommand()`. The command will not be registered with the server and will be non-functional. The server's command system is solely responsible for its lifecycle.
- **External Invocation:** Do not call the execute method directly from other game systems. This bypasses critical infrastructure for argument parsing, permission checks, and context setup. To trigger command logic programmatically, use the server's command execution service.
- **Reliance on Output:** The command's output is a temporary, client-side visual selection. Systems should not be built with the assumption that this selection is a permanent or server-authoritative world change.

## Data Pipeline
The flow of data for this command is a multi-stage process, transforming a text command into a complex in-world visualization.

> Flow:
> Player Chat Input -> Server Command Parser -> **BlockSelectCommand.execute()** -> BlockTypeAssetMap Query -> Parallel Stream Filter & Sort -> Grid Layout Calculation -> SelectionProvider.computeSelectionCopy -> Client-Side Selection Render

