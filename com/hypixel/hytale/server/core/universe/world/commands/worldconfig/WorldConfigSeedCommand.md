---
description: Architectural reference for WorldConfigSeedCommand
---

# WorldConfigSeedCommand

**Package:** com.hypixel.hytale.server.core.universe.world.commands.worldconfig
**Type:** Transient / Stateless Service

## Definition
```java
// Signature
public class WorldConfigSeedCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The WorldConfigSeedCommand is a concrete implementation of the Command Pattern, designed to operate within the server's command processing system. It serves a single, specific function: to retrieve the current world's seed and report it to the command's issuer.

By extending AbstractWorldCommand, this class integrates seamlessly into the server's command registry. The registry is responsible for discovering, storing, and dispatching commands based on user input. This class acts as a terminal node in the command execution chain, translating a high-level user request ("/seed") into a low-level data retrieval operation against the active World object. Its existence decouples the command parsing and routing logic from the business logic of accessing world configuration.

## Lifecycle & Ownership
- **Creation:** Instantiated once by the server's command registration system during the initial bootstrap phase. The system typically scans the classpath for all subclasses of AbstractWorldCommand and creates a singleton instance of each to populate its registry.
- **Scope:** Application-level. The single instance persists for the entire duration of the server session. It is not tied to a specific world or player session.
- **Destruction:** The instance is dereferenced and eligible for garbage collection only upon server shutdown when the central command registry is cleared.

## Internal State & Concurrency
- **State:** This class is fundamentally **stateless and immutable**. It contains no member fields to store data between invocations. All necessary state, such as the target World and the CommandContext, is passed as arguments to the execute method.
- **Thread Safety:** The class is inherently thread-safe. As it holds no state, concurrent invocations of the execute method cannot interfere with one another. Responsibility for thread-safe access to the World and CommandContext objects lies with the calling Command System, which must guarantee that these objects are not mutated by other threads during the command's execution.

## API Surface
The public contract is defined by its constructor for registration and the overridden execute method for invocation by the framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| WorldConfigSeedCommand() | constructor | O(1) | Registers the command with the name "seed" and a translatable description key. |
| execute(context, world, store) | void | O(1) | Executes the command's logic. Retrieves the seed from the provided World object and sends it as a message via the CommandContext. |

## Integration Patterns

### Standard Usage
A developer or user does not interact with this class directly. The server's command system invokes it in response to parsed user input. The conceptual flow is managed entirely by the framework.

```java
// CONCEPTUAL: How the Command System invokes this class.
// This code is not intended for direct use by developers.

// 1. User types "/seed"
String userInput = "/seed";

// 2. System parses and finds the corresponding command
AbstractWorldCommand command = commandRegistry.find("seed");

// 3. System prepares context and executes
if (command instanceof WorldConfigSeedCommand) {
    CommandContext context = createCommandContextForUser(...);
    World currentWorld = server.getCurrentWorld();
    Store<EntityStore> entityStore = currentWorld.getEntityStore();
    command.execute(context, currentWorld, entityStore);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new WorldConfigSeedCommand()`. The command will not be registered with the server's command system and will never be executed in response to user input. The framework handles instantiation.
- **Direct Invocation:** Avoid calling the `execute` method directly. Doing so bypasses the command system's infrastructure for permission checks, context creation, and error handling. The CommandContext and World arguments must be supplied by the framework to be valid.

## Data Pipeline
The data flow for this command is simple and unidirectional, originating from user input and terminating as a message sent back to the user.

> Flow:
> User Input (`/seed`) -> Network Layer -> Command Parser -> Command Registry -> **WorldConfigSeedCommand.execute()** -> World.getWorldConfig() -> Message -> CommandContext.sendMessage() -> Network Layer -> Client Chat UI

