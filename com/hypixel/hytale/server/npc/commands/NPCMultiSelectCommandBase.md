---
description: Architectural reference for NPCMultiSelectCommandBase
---

# NPCMultiSelectCommandBase

**Package:** com.hypixel.hytale.server.npc.commands
**Type:** Transient Framework Component

## Definition
```java
// Signature
public abstract class NPCMultiSelectCommandBase extends NPCWorldCommandBase {
```

## Architecture & Concepts
The NPCMultiSelectCommandBase is an abstract base class that serves as a foundational component for server-side commands requiring the selection of one or more Non-Player Characters (NPCs). It encapsulates the complex logic of entity selection based on spatial queries and attribute filtering, providing a standardized framework for developers to build upon.

This class acts as a bridge between the server's command system and the Entity-Component-System (ECS) world state. Its primary responsibility is to parse player-provided arguments—such as range, viewing angle (cone), and NPC roles—and translate them into a precise query against the world's EntityStore. It uses utility classes like TargetUtil to perform the actual spatial calculations (ray-casting, sphere-casting) and then applies further filtering logic.

By inheriting from this class, concrete command implementations (e.g., a command to modify an NPC's behavior or appearance) are absolved from writing boilerplate selection code. They only need to implement the logic for what to *do* with the list of selected NPCs, which is provided to them by the base class. This promotes code reuse and ensures a consistent user experience for targeting NPCs across different commands.

### Lifecycle & Ownership
-   **Creation:** Concrete subclasses are instantiated once by the server's command registration system during server initialization or plugin loading. These singleton instances are registered and mapped to their corresponding command names.
-   **Scope:** The command object instance persists for the entire server runtime. However, the execution of its logic is transient; a new execution context is established for each command invocation and is discarded upon completion.
-   **Destruction:** The object is garbage collected when the server shuts down or when the plugin providing the command is unloaded, effectively de-registering it from the command system.

## Internal State & Concurrency
-   **State:** This class is effectively stateless with respect to command execution. It defines several final fields (e.g., coneAngleArg, rangeArg) which are immutable argument definitions. All state required for a command's execution, such as the command sender and world state, is passed as parameters to the execute method. This design ensures that a single command instance can be safely reused for concurrent executions by different players.
-   **Thread Safety:** The class is thread-safe. The execute method is designed to be called by the server's main command processing thread. It operates exclusively on method-local variables and the provided parameters. All interactions with the game world (via the Store object) are expected to be handled by the engine's underlying concurrency model, typically by ensuring command execution occurs on the main game thread.

## API Surface
The primary interaction points are the overridden entry point and the method subclasses must implement to process the results.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | protected void | O(N) | Overridden entry point. Contains the core selection logic. N is the number of entities within the initial coarse query range. **WARNING:** Do not override this method in subclasses unless you intend to fundamentally change the selection mechanism. |
| processEntityList(context, world, store, refs) | protected void | O(M) | Processes the final list of selected entities. M is the number of entities that passed all filters. By default, it iterates and calls the single-entity execute method for each. |

## Integration Patterns

### Standard Usage
A developer should extend NPCMultiSelectCommandBase and implement the single-entity `execute` method inherited from a parent class. The base class will handle argument parsing, entity selection, and filtering, then invoke the subclass's logic for each selected NPC via the `processEntityList` method.

```java
// Example: A command to make selected NPCs jump
public class NPCJumpCommand extends NPCMultiSelectCommandBase {

    public NPCJumpCommand() {
        super("jump", "Makes the selected NPC(s) jump.");
    }

    // This is the method where the action is defined.
    // It is called by the base class's processEntityList for each selected NPC.
    @Override
    protected void execute(@Nonnull CommandContext context, @Nonnull NPCEntity npc, @Nonnull World world, @Nonnull Store<EntityStore> store, @Nonnull Ref<EntityStore> ref) {
        // Apply jump logic to the 'npc'
        // ...
        context.sendMessage(Message.text("Made " + npc.getName() + " jump!"));
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance of a command using `new NPCMyCommand()`. Commands must be registered with the server's command system, which manages their lifecycle.
-   **Overriding Core Execution:** Avoid overriding the primary `execute(context, world, store)` method. Doing so bypasses the built-in selection and filtering logic, defeating the purpose of using this base class.
-   **Stateful Subclasses:** Do not add mutable instance variables to your command subclass. This breaks thread safety and can lead to unpredictable behavior when multiple players execute the command simultaneously.

## Data Pipeline
The flow of data and logic for a typical command execution is linear and deterministic, transforming high-level player input into specific world actions.

> Flow:
> Player Command String -> Server Command Parser -> **NPCMultiSelectCommandBase.execute()** -> Argument Parsing (Range, Angle, Roles) -> Broad-phase Spatial Query (TargetUtil.getAllEntitiesInSphere) -> Fine-phase Filtering (Cone check, Role check) -> Sorting (Nearest check) -> `List<Ref<EntityStore>>` -> **processEntityList()** -> Subclass Logic per Entity

