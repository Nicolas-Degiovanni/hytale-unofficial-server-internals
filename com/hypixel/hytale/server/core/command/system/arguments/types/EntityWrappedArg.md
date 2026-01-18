---
description: Architectural reference for EntityWrappedArg
---

# EntityWrappedArg

**Package:** com.hypixel.hytale.server.core.command.system.arguments.types
**Type:** Transient

## Definition
```java
// Signature
public class EntityWrappedArg extends WrappedArg<UUID> {
```

## Architecture & Concepts
The EntityWrappedArg is a specialized argument resolver within the server's command processing framework. Its primary function is to translate a command argument into a valid, world-aware reference to an entity, represented by a `Ref<EntityStore>`.

This class embodies the **Strategy Pattern** for entity resolution. It decouples the high-level command execution logic from the low-level mechanics of finding an entity. It supports two distinct resolution strategies:

1.  **Direct Resolution:** The command provides an explicit entity UUID. EntityWrappedArg validates this UUID and retrieves the corresponding entity from the world.
2.  **Implicit Resolution:** The command provides no entity argument. In this case, the resolver uses the command sender's context (specifically, their line of sight) to determine the target entity.

This dual-strategy approach provides significant flexibility for command designers, allowing for both precise administrative commands and intuitive, context-aware player commands.

## Lifecycle & Ownership
- **Creation:** An EntityWrappedArg instance is created during the server's command registration phase. It is constructed by the command definition builder, wrapping a more primitive argument type that is responsible for parsing a UUID from the raw command string. It is not created per-execution, but once per command definition.
- **Scope:** The object's lifetime is bound to the command it is associated with. It persists as part of the command's definition within the server's command registry.
- **Destruction:** It is eligible for garbage collection when the associated command is unregistered or when the server shuts down and clears the command registry.

## Internal State & Concurrency
- **State:** The class is effectively immutable after construction. It holds a final reference to the underlying `Argument` but does not mutate its own state during resolution calls. All resolution logic depends on the transient `CommandContext` and `World` state passed into its methods.
- **Thread Safety:** **This class is not thread-safe.** All interactions with this class must occur on the main server thread. Its methods operate directly on core game state objects like `World` and `EntityStore`, which are not designed for concurrent access. Calling its methods from an asynchronous task or a different thread will lead to critical data corruption and server instability.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(accessor, context) | Ref<EntityStore> | Variable | The primary resolution method. Attempts direct UUID lookup first, falling back to line-of-sight targeting. Throws `GeneralCommandException` on failure. |
| getEntityDirectly(context, world) | Ref<EntityStore> | O(1) | Attempts to resolve an entity using only an explicitly provided UUID argument. Returns null if no argument was provided. Throws `GeneralCommandException` if the UUID is invalid. |

## Integration Patterns

### Standard Usage
EntityWrappedArg is intended to be used declaratively within a command's definition. The command's execution logic receives the resolved `Ref<EntityStore>` after the argument parsing and resolution phase is complete.

```java
// Within a command's execution block
public void execute(CommandContext context) {
    // The 'entity' argument is an instance of EntityWrappedArg
    EntityWrappedArg entityArg = context.getArgument("target_entity", EntityWrappedArg.class);

    // The 'get' call performs the complex resolution logic
    Ref<EntityStore> targetEntityRef = entityArg.get(this.componentAccessor, context);

    // Perform actions on the validated entity reference
    if (targetEntityRef.isValid()) {
        // ... apply effect to targetEntityRef
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Instantiation:** Do not use `new EntityWrappedArg()`. This class should only be created as part of a command builder's fluent interface to ensure it is correctly registered with the command system.
- **Off-Thread Execution:** Never call the `get` method from any thread other than the main server tick thread. This will bypass all thread-safety mechanisms for world state.
- **Ignoring Exceptions:** The `get` method throws `GeneralCommandException` for user-facing errors (e.g., "Player not found"). This exception must be caught and propagated to the command sender; failure to do so will result in unhandled exception noise in the server console.

## Data Pipeline
The EntityWrappedArg serves as a critical transformation step in the command processing pipeline, converting a low-level identifier into a high-level, safe game object reference.

> Flow:
> Raw Command String -> Command Parser -> `Argument<UUID>` (Parses UUID) -> **EntityWrappedArg** (Resolves UUID or line-of-sight) -> `Ref<EntityStore>` -> Command Executor Logic

