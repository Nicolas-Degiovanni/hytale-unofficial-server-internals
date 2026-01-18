---
description: Architectural reference for DebugShapeConeCommand
---

# DebugShapeConeCommand

**Package:** com.hypixel.hytale.server.core.modules.debug.commands
**Type:** Service

## Definition
```java
// Signature
public class DebugShapeConeCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The DebugShapeConeCommand is a concrete implementation of the **Command Pattern**, designed to integrate with the server's central command processing system. It serves as a terminal endpoint for player-initiated chat commands, specifically for in-game debugging and visualization.

Its primary architectural role is to act as a bridge between the Command System and the Debug Rendering System. When invoked, it queries the server's Entity Component System (ECS) to retrieve the position of the executing player via their TransformComponent. It then uses this positional data to instruct the DebugUtils service to render a primitive shape (a cone) in the game world.

This class is intended exclusively for development and testing environments. It provides a direct, in-game mechanism for developers to trigger visual debugging aids without requiring external tools or client-side modifications.

### Lifecycle & Ownership
- **Creation:** A single instance is created by the server's command registration service during the server bootstrap sequence. The system scans for command implementations and instantiates them for its internal registry.
- **Scope:** The instance is a singleton within the scope of the command registry. It persists for the entire lifecycle of the server session.
- **Destruction:** The object is dereferenced and eligible for garbage collection only when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is **stateless**. It contains no mutable instance fields. All required data, such as the world reference, player entity, and command context, is provided as method parameters during the execution call. The only internal field is a static, immutable reference to a translatable message string.
- **Thread Safety:** This class is **conditionally thread-safe**. The instance itself can be safely shared across threads because it has no mutable state. However, the execute method is designed to be called by the server's main game thread, which has exclusive write access to the game state (World, EntityStore). Invoking execute from a worker thread would violate the server's threading model and lead to race conditions and data corruption. The use of ThreadLocalRandom ensures that random number generation is safe within the context of the calling thread.

## API Surface
The public contract is defined by its superclass, AbstractPlayerCommand. The core logic resides in the protected execute method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DebugShapeConeCommand() | constructor | O(1) | Instantiates the command and registers its name and description with the superclass. |
| execute(...) | protected void | O(1) | Executes the primary command logic. Fetches player position and delegates to DebugUtils to render a cone. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developer code. It is used exclusively by the server's command dispatch system. A user with appropriate permissions triggers its execution by typing the command into the game client's chat console.

*Example of the server-side dispatch flow (conceptual):*
```java
// The Command System finds and executes the registered command
// This code does NOT appear in typical game logic
Command command = commandRegistry.find("cone");
if (command instanceof AbstractPlayerCommand) {
    ((AbstractPlayerCommand) command).execute(context, store, ref, playerRef, world);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new DebugShapeConeCommand()`. The command system is responsible for the object's lifecycle. Manually creating an instance will result in a disconnected object that is not registered to handle any chat input.
- **External Invocation:** Do not call the `execute` method directly from other game systems. This bypasses the command system's critical infrastructure, including permission checks, argument parsing, and context setup. If you need to draw a debug shape, use the DebugUtils service directly.

## Data Pipeline
The flow of data for this command begins with user input and terminates with a visual change in the game world for all nearby clients.

> Flow:
> Player Chat Input (`/cone`) -> Server Network Listener -> Command Parser -> Command Dispatcher -> **DebugShapeConeCommand.execute()** -> ECS Query (for TransformComponent) -> DebugUtils.addCone() -> World State Update -> Network Packet (to clients) -> Client-Side Debug Renderer

---

