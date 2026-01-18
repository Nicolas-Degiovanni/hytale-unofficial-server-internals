---
description: Architectural reference for ObjectiveCommand
---

# ObjectiveCommand

**Package:** com.hypixel.hytale.builtin.adventure.objectives.commands
**Type:** Transient

## Definition
```java
// Signature
public class ObjectiveCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The ObjectiveCommand class serves as a **Command Aggregator** for all objective-related server commands. It does not implement any game logic itself. Instead, its sole responsibility is to act as a namespace and router, grouping various sub-commands under the primary `/objective` command (and its alias `/obj`).

This class extends AbstractCommandCollection, inheriting the framework necessary to register a collection of child commands. Each child, such as ObjectiveStartCommand or ObjectiveCompleteCommand, encapsulates the specific logic for one action. This design adheres to the **Command Pattern** and promotes high cohesion and low coupling; the logic for starting an objective is completely separate from the logic for viewing its history.

In the server's command processing pipeline, ObjectiveCommand is an intermediary node. When a player executes a command like `/objective start my_quest`, the central command system first identifies and routes the request to the ObjectiveCommand instance. ObjectiveCommand then parses the next argument ("start") and delegates the remainder of the command context to the corresponding registered sub-command.

### Lifecycle & Ownership
-   **Creation:** A single instance of ObjectiveCommand is created by the server's command registration system during the server bootstrap sequence. It is not intended for manual instantiation during gameplay.
-   **Scope:** The instance is registered with the server's central CommandRegistry and persists for the entire server session. It is effectively a stateless singleton managed by the command framework.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server is shutting down and the CommandRegistry is cleared.

## Internal State & Concurrency
-   **State:** This class is **stateless and effectively immutable** after its constructor completes. Its internal state consists of a list of sub-command instances, which is populated once and never modified during runtime.
-   **Thread Safety:** The class is inherently **thread-safe**. Its immutable nature allows multiple threads from the server's command execution pool to safely traverse its sub-command list without locks or synchronization. All state is established before the command becomes available for execution by players.

## API Surface
The public contract of this class is primarily its constructor, which is used by the framework to build the command group. It has no other public methods intended for external invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ObjectiveCommand() | constructor | O(k) | Constructs the command aggregator, where k is the fixed number of sub-commands. Populates the internal sub-command registry. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly. Interaction occurs by executing the command in-game or by creating a new sub-command and registering it within the constructor.

The following example demonstrates how a new, hypothetical sub-command would be added to this collection during development.

```java
// Example: Adding a new "pause" sub-command
// This modification would be made directly to the ObjectiveCommand.java file.

public class ObjectiveCommand extends AbstractCommandCollection {
   public ObjectiveCommand() {
      super("objective", "server.commands.objective");
      this.addAliases("obj");
      this.addSubCommand(new ObjectiveStartCommand());
      this.addSubCommand(new ObjectiveCompleteCommand());
      // ... other commands
      
      // New sub-command is added here
      this.addSubCommand(new ObjectivePauseCommand()); 
   }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance of ObjectiveCommand using `new ObjectiveCommand()` in game logic. The command system handles its lifecycle. Doing so would create an unregistered, non-functional command object.
-   **Runtime Modification:** Do not attempt to retrieve this object from the command registry and add sub-commands at runtime. The command collection is intended to be sealed at server startup. Modifying it later can lead to race conditions and unpredictable behavior.

## Data Pipeline
ObjectiveCommand functions as a routing step in the server's command processing data flow. It receives a command context and dispatches it to the appropriate handler based on the first argument.

> Flow:
> Player Chat Input (`/objective ...`) -> Server Network Listener -> Command Parsing Service -> **ObjectiveCommand (Router)** -> Matched Sub-Command (e.g., ObjectiveStartCommand) -> Objective Management System -> Player Feedback


