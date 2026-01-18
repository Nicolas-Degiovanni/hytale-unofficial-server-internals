---
description: Architectural reference for LightingSendToggleCommand
---

# LightingSendToggleCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.lighting
**Type:** Abstract Command Template

## Definition
```java
// Signature
abstract class LightingSendToggleCommand extends AbstractWorldCommand {
```

## Architecture & Concepts
The LightingSendToggleCommand is an abstract base class that provides a reusable template for creating server commands that toggle a boolean state. It is a foundational component within the server's command system, designed to eliminate boilerplate code for simple on/off or enable/disable functionality.

This class employs the **Template Method Pattern**. The `execute` method defines a fixed algorithm for the toggle logic:
1.  Parse an optional boolean argument from the command context.
2.  If the argument is absent, invert the current state.
3.  If the argument is present, use its value directly.
4.  Apply the new state.
5.  Report the result to the command issuer.

Crucially, this class is decoupled from the state it manipulates. It does not know *what* it is toggling, only *how* to toggle it. This is achieved by requiring concrete subclasses to provide a `BooleanSupplier` (getter) and a `Consumer<Boolean>` (setter) during construction. This functional approach allows the same command logic to be applied to any boolean property within the server, such as world lighting flags, feature toggles, or debug settings.

By extending AbstractWorldCommand, it integrates directly into the server's command processing pipeline, ensuring it is executed with the necessary World and EntityStore context.

## Lifecycle & Ownership
-   **Creation:** Concrete subclasses of LightingSendToggleCommand are instantiated once during server initialization by the command registration system. They are not created on a per-request basis.
-   **Scope:** These command objects are singletons that persist for the entire lifetime of the server. A single instance of a concrete command like `ToggleSkyLightCommand` handles all invocations of that command.
-   **Destruction:** The object is garbage collected when the server shuts down and the command registry is cleared. It requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** This class is **stateless and immutable**. Its internal fields, including the getter and setter functions, are final and are assigned only during construction. It does not store or cache the boolean value it controls; it queries the authoritative source via the injected `getter` on every execution.

-   **Thread Safety:** The class itself is inherently thread-safe. However, the overall safety of the operation is **entirely dependent on the provided getter and setter functions**. If these functions access a shared resource (e.g., a world configuration object) that is not thread-safe, race conditions could occur.

    **Warning:** The server's command system typically executes commands sequentially on the main server thread. As long as the injected functions do not spawn new threads, operations are generally safe. Developers must ensure that the state being modified by the setter is managed in a thread-safe manner if it can be accessed by other systems.

## API Surface
The primary contract is the `protected` constructor used by subclasses. The public-facing contract is the `execute` method inherited from the command system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, store) | void | O(1) | Executes the toggle logic. Throws CommandException if argument parsing fails. The complexity of the injected getter and setter is external to this class. |

## Integration Patterns

### Standard Usage
A developer must extend this class to create a new toggle command. The `super` constructor must be called with the command's metadata and, most importantly, method references or lambdas for getting and setting the target boolean value.

```java
// A concrete implementation for toggling a hypothetical "sunlight" setting.
public class ToggleSunlightCommand extends LightingSendToggleCommand {

    // The settings object is injected from a central service.
    public ToggleSunlightCommand(WorldLightingSettings settings) {
        super(
            "togglesunlight",
            "Enables or disables global sunlight.",
            "The desired state for sunlight.",
            "commands.lighting.sunlight.status",
            settings::isSunlightEnabled,  // Getter: A method reference to the state source
            settings::setSunlightEnabled  // Setter: A method reference to mutate the state
        );
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Complex Logic in Lambdas:** The provided getter and setter functions should be simple accessors. Do not embed complex business logic, network calls, or other command executions within them. This violates the single responsibility principle and makes the command difficult to debug.
-   **Stateful Lambdas:** Avoid providing lambdas that capture and modify mutable local variables. The getter and setter must operate on a stable, centrally managed service or state object.
-   **Bypassing the Setter:** Do not modify the underlying state from another part of the command and then call the setter. The setter should be treated as the sole entry point for mutation within this command's lifecycle.

## Data Pipeline
The flow for this component is initiated by a user and results in a state change and a feedback message.

> Flow:
> User Input (`/togglesunlight false`) -> Server Command Parser -> Command Dispatcher -> **LightingSendToggleCommand.execute()** -> Injected `setter` (`WorldLightingSettings.setSunlightEnabled`) -> World State Update
>
> Feedback Loop:
> **LightingSendToggleCommand.execute()** -> `CommandContext.sendMessage()` -> Server Network Layer -> Client Chat UI

