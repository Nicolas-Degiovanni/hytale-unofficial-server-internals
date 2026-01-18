---
description: Architectural reference for NPCCommand
---

# NPCCommand

**Package:** com.hypixel.hytale.server.npc.commands
**Type:** Transient

## Definition
```java
// Signature
public class NPCCommand extends AbstractCommandCollection {
```

## Architecture & Concepts

The NPCCommand class serves as the root command aggregator for all Non-Player Character (NPC) related server commands. It does not implement any game logic itself; instead, it acts as a dispatcher, routing command invocations to a collection of specialized sub-commands like NPCSpawnCommand, NPCDebugCommand, and NPCPathCommand. This design follows the **Command Pattern**, centralizing the registration and structure of the entire `/npc` command family.

The most significant architectural component within this class is the static final field **NPC_ROLE**. This is a custom argument parser, an implementation of SingleArgumentType, designed to bridge the gap between raw user string input and the server's internal NPC data structures.

Key responsibilities of NPC_ROLE include:
*   **Parsing:** It converts a string identifier (e.g., "guard") into a structured BuilderInfo object.
*   **Validation:** It queries the central NPCPlugin to verify if the provided role name exists.
*   **Error Handling:** It generates localized, user-friendly error messages, including "did you mean" suggestions for typos, by performing fuzzy string matching against the list of available roles.
*   **Tab Completion:** It provides real-time suggestions to the command sender as they type, enhancing usability.

By encapsulating this logic, NPC_ROLE ensures that any command requiring an NPC role as a parameter benefits from consistent, robust, and user-friendly argument handling without duplicating code.

## Lifecycle & Ownership

-   **Creation:** An instance of NPCCommand is created by the server's core command system during the plugin loading phase. It is not intended for manual instantiation by developers.
-   **Scope:** The object's lifecycle is tied to the server session. Once registered, it persists in the command map until the server or the parent NPCPlugin is shut down.
-   **Destruction:** The instance is marked for garbage collection when the server unloads the command registry, typically during a server shutdown sequence.

## Internal State & Concurrency

-   **State:** The NPCCommand instance is effectively immutable after construction. Its state consists of the list of registered sub-commands, which is populated exclusively within the constructor and is not modified thereafter. The static NPC_ROLE field is a stateless definition of behavior.
-   **Thread Safety:** This class is not thread-safe by design and expects to be accessed only from the main server thread, which is the standard execution model for the command system. The NPC_ROLE argument parser delegates calls to the NPCPlugin singleton. Its thread safety is therefore dependent on the thread safety of NPCPlugin.

    **Warning:** Any modifications to this class or its dependencies must assume a single-threaded execution context within the server's main game loop.

## API Surface

The primary public contract is not a method but the static argument type used for integration with other commands.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| NPC_ROLE | SingleArgumentType | O(N) | A static argument parser that resolves a string role name into a BuilderInfo object. Complexity is O(N) where N is the number of registered NPC roles, due to lookup and fuzzy matching for suggestions. |

## Integration Patterns

### Standard Usage

The primary integration pattern is to use the static NPC_ROLE field when defining a command that requires the user to specify an NPC type. This ensures consistent parsing and user experience across all NPC-related commands.

```java
// Example of a new subcommand using the NPC_ROLE argument type
CommandNode.builder("setrole")
    .argument("target_npc", EntityArgument.entity())
    .argument("role_name", NPCCommand.NPC_ROLE) // Correct usage
    .executes(context -> {
        // Retrieve the parsed BuilderInfo object
        BuilderInfo info = context.getArgument("role_name", BuilderInfo.class);
        Entity target = context.getArgument("target_npc", Entity.class);

        // Apply the role information to the target entity
        // ...

        return Command.SUCCESS;
    })
    .build();
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not call `new NPCCommand()`. This creates an orphaned object that is not registered with the server's command system and serves no purpose.
-   **Manual Role Parsing:** Avoid manually parsing role strings by calling `NPCPlugin.get()` directly within a command's execution logic. This bypasses the centralized validation, error handling, and suggestion logic provided by NPC_ROLE, leading to inconsistent behavior and code duplication.

## Data Pipeline

The data flow for parsing an NPC role argument demonstrates how NPCCommand decouples user input from the game system.

> Flow:
> User Input String (e.g., "guard") → Server Command Dispatcher → **NPC_ROLE.parse("guard")** → NPCPlugin.getIndex("guard") → NPCPlugin.getRoleBuilderInfo(index) → BuilderInfo Object → Command Execution Context

