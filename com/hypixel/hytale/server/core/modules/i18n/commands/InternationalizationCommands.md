---
description: Architectural reference for InternationalizationCommands
---

# InternationalizationCommands

**Package:** com.hypixel.hytale.server.core.modules.i18n.commands
**Type:** Transient

## Definition
```java
// Signature
public class InternationalizationCommands extends AbstractCommandCollection {
```

## Architecture & Concepts
The InternationalizationCommands class serves as a structural container within the server's command system. It is not a command that executes logic itself; rather, it acts as a top-level namespace or group for a collection of related subcommands.

Its primary architectural function is to aggregate all internationalization (i18n) related server commands under a single, user-facing entry point: **lang**. By extending AbstractCommandCollection, it integrates seamlessly into the command registration and dispatching pipeline, allowing for a clean, hierarchical command structure. This pattern simplifies command discovery for both administrators and the underlying system by grouping functionality.

In this specific implementation, it registers the **lang** command and its aliases, and attaches the GenerateI18nCommand as its sole subcommand.

### Lifecycle & Ownership
-   **Creation:** An instance of InternationalizationCommands is created by the server's command registration system during the server bootstrap or module loading phase. It is discovered and instantiated reflectively or via a dedicated registry.
-   **Scope:** The object's lifetime is tied to the server session. It is created once upon startup and persists until server shutdown. It is held as a reference within the central CommandManager or an equivalent service.
-   **Destruction:** The object is eligible for garbage collection upon server shutdown when the command registry is cleared. It manages no native resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** The state of this object is effectively immutable after construction. The constructor initializes the command name, description, aliases, and the list of subcommands. There are no public methods to mutate this state post-instantiation.
-   **Thread Safety:** This class is inherently thread-safe. Its state is established in the constructor on the main server thread during initialization. Subsequent reads of its configuration (name, aliases, subcommands) by the command dispatcher from multiple player threads are safe due to the immutable nature of the data.

## API Surface
The public contract is entirely defined by the constructor and the inherited methods from AbstractCommandCollection, which are consumed by the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| InternationalizationCommands() | Constructor | O(1) | Initializes the command group with the name "lang", its aliases, and registers the GenerateI18nCommand as a subcommand. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by module or plugin developers. The server's command system automatically discovers and registers it. A server administrator interacts with the *result* of this class via the game console.

```bash
# Example of an administrator invoking the subcommand
# registered by this collection.
/lang generate
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance of this class manually. Doing so will have no effect, as the instance will not be registered with the server's central command dispatcher and will be immediately garbage collected.
-   **State Modification:** Do not attempt to use reflection to modify the internal list of subcommands after the object has been constructed and registered. The command system does not expect or support dynamic changes to command collections and such actions will lead to undefined behavior.

## Data Pipeline
This class primarily acts as a routing definition within the command processing pipeline. It does not transform data itself.

> Flow:
> Player Command Input (`/lang generate`) -> Network Layer -> Command Dispatcher -> **InternationalizationCommands** (Route Match) -> GenerateI18nCommand (Execution) -> Command Result -> Network Layer -> Player Console Output

