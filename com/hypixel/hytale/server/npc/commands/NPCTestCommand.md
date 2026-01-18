---
description: Architectural reference for NPCTestCommand
---

# NPCTestCommand

**Package:** com.hypixel.hytale.server.npc.commands
**Type:** Definition

## Definition
```java
// Signature
public class NPCTestCommand extends AbstractCommandCollection {
   // ...
   public static class ProbeTestCommand extends AbstractPlayerCommand {
      // ...
   }
}
```

## Architecture & Concepts
The NPCTestCommand class serves as a registration point for a collection of server-side diagnostic commands related to the Non-Player Character (NPC) system. It is not a service or a manager, but rather a definition that plugs into the server's core CommandSystem.

This class follows the **Composite Command** pattern. The top-level NPCTestCommand acts as a container for more specific sub-commands, such as the inner class ProbeTestCommand. This structure allows for a clean command hierarchy, for example, `/npc test probe`.

The primary architectural role of this command is to provide a live, in-game debugging interface for complex server-side systems. Specifically, the ProbeTestCommand sub-command directly invokes and stress-tests the collision and position validation logic used by the NPC spawning and pathfinding systems. It retrieves the executing player's real-time entity data (position, bounding box) and uses it as input for various position probing utilities (PositionProbeAir, PositionProbeWater) and the global CollisionModule. The results are then fed back to the administrator, providing immediate insight into how the server evaluates spatial validity.

### Lifecycle & Ownership
-   **Creation:** A single instance of NPCTestCommand is created by the NPCPlugin during its bootstrap sequence. The plugin is responsible for registering this command collection with the server's central CommandSystem.
-   **Scope:** The object instance persists for the entire lifecycle of the NPCPlugin. It is effectively a static definition that exists as long as the server is running and the plugin is enabled.
-   **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the NPCPlugin is unloaded or the server shuts down. The CommandSystem manages the removal of the command registration.

## Internal State & Concurrency
-   **State:** This class is **stateless**. It holds no mutable fields and does not cache data between invocations. All necessary context, such as the player entity and world state, is provided as method-local parameters to the `execute` method by the CommandSystem. Utility objects like PositionProbeAir are instantiated on the stack for each execution, ensuring no state is shared.
-   **Thread Safety:** The server's CommandSystem guarantees that all command `execute` methods are invoked exclusively on the main server thread. Consequently, this class is inherently thread-safe, as no concurrent access is possible. All interactions with the world and the Entity Component System (ECS) Store are safe within this execution context.

## API Surface
The primary interface for this class is not programmatic but rather the in-game command syntax it defines. The core logic resides in the `execute` method of its sub-commands.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ProbeTestCommand.execute(...) | void | O(N) | Executes the position validation tests from the perspective of the command-issuing player. Complexity is dominated by the call to CollisionModule.validatePosition, which performs a spatial query against world geometry. N is the number of potential colliders in the queried area. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is designed to be registered by its parent plugin with the server's CommandSystem. A player with sufficient permissions then invokes it via the in-game chat console.

The registration process within the parent plugin would look like this:
```java
// Inside NPCPlugin.java during initialization
CommandSystem commandSystem = server.getService(CommandSystem.class);
if (commandSystem != null) {
    commandSystem.registerCommand(new NPCTestCommand());
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation and Execution:** Do not manually create an instance of ProbeTestCommand and call its `execute` method. Doing so bypasses the CommandSystem's critical context injection (providing the World, Store, PlayerRef) and permission checks, which will result in immediate NullPointerExceptions and a server crash.
-   **Stateful Modification:** Do not modify this class to hold state between executions. Commands must be stateless to ensure predictable behavior and prevent memory leaks, especially on a server with many concurrent players.

## Data Pipeline
The NPCTestCommand acts as a user-triggered entry point into the server's core physics and world state systems. The data flow is initiated by a player and results in feedback to that same player.

> Flow:
> Player Chat Input (`/npc test probe`) -> Client Network Packet -> Server Network Layer -> Command Parser -> **NPCTestCommand.ProbeTestCommand.execute()** -> ECS Store & World Query -> CollisionModule -> Message Object Creation -> Server Network Layer -> Client Chat Output & Server Log File

