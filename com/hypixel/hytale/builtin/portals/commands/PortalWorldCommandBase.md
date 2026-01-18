---
description: Architectural reference for PortalWorldCommandBase
---

# PortalWorldCommandBase

**Package:** com.hypixel.hytale.builtin.portals.commands
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class PortalWorldCommandBase extends AbstractWorldCommand {
```

## Architecture & Concepts
PortalWorldCommandBase is a foundational component within the server-side command system, designed to act as a secure and standardized template for all commands that must operate exclusively within the context of a PortalWorld.

This class implements the **Template Method design pattern**. The primary `execute` method is marked as `final`, establishing a non-overridable execution algorithm. This algorithm first validates that the command is being executed within a world that contains a valid PortalWorld resource. Only after this critical precondition is met does it delegate control to the abstract `execute` method, which must be implemented by concrete subclasses.

By centralizing this validation logic, PortalWorldCommandBase provides a critical architectural safeguard. It prevents a wide class of runtime errors by ensuring that portal-specific logic can never be invoked in an incorrect or invalid world context. Subclasses are freed from the boilerplate of resource fetching and validation, allowing them to focus solely on their specific business logic.

### Lifecycle & Ownership
- **Creation:** Concrete subclasses are instantiated once by the server's command registration system during the bootstrap or module loading phase. The command system is the sole owner of these command instances.
- **Scope:** A single command instance is created and persists for the entire lifetime of the server. The class is inherently stateless, allowing one instance to safely handle all concurrent invocations.
- **Destruction:** The instance is marked for garbage collection when the server shuts down and the central command registry is cleared.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no mutable fields. Its behavior is determined entirely by the arguments provided during a command invocation, primarily the CommandContext and the World.
- **Thread Safety:** The class itself is unconditionally thread-safe. However, command execution is managed by the server's command system, which guarantees that each command is executed on the appropriate world's main thread. This prevents race conditions and concurrent modification issues related to the World and its associated Store. Subclasses should not fork new threads that modify world state without extreme care.

## API Surface
The public contract is defined by the abstract method that subclasses must implement. The user-facing command invocation is handled by the parent AbstractWorldCommand class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(context, world, portalWorld, store) | void | Subclass-dependent | Abstract method for subclass implementation. It is guaranteed to be invoked only within a valid PortalWorld context. The validated PortalWorld resource is passed directly as an argument. |

## Integration Patterns

### Standard Usage
Developers should extend this class to create any new command that requires interaction with PortalWorld data. The implementation should focus exclusively on the logic within the overridden `execute` method.

```java
// A concrete command that modifies a portal's destination.
public class SetPortalDestinationCommand extends PortalWorldCommandBase {

    public SetPortalDestinationCommand() {
        super("setportaldest", "Sets the destination for the current portal.");
        // Define arguments for the command here.
    }

    @Override
    protected void execute(@Nonnull CommandContext context, @Nonnull World world, @Nonnull PortalWorld portalWorld, @Nonnull Store<EntityStore> store) {
        // The portalWorld object is guaranteed to exist and be valid.
        // Retrieve destination from command arguments.
        String newDestination = context.getArgument("destination");

        portalWorld.setTarget(newDestination);
        context.sendMessage(Message.text("Portal destination updated."));
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Validation:** Do not re-fetch or re-validate the PortalWorld resource inside your subclass. The base class contract guarantees it exists. Redundant checks add unnecessary overhead.
- **Bypassing the Template:** Do not attempt to override the `final execute` method via reflection or other means. Doing so would break the critical safety guarantees provided by the base class.
- **Direct Instantiation:** Command objects should never be instantiated with `new` outside of the command registration process. The command system manages their lifecycle.

## Data Pipeline
The flow for any command inheriting from this class follows a strict, validation-first sequence.

> Flow:
> Player Command Input -> Network Layer -> Command Parser -> **PortalWorldCommandBase (Template Method)** -> Precondition Check: `store.getResource(PortalWorld.class)` -> **Concrete Subclass Implementation** -> World State Mutation or Query -> Response Message -> Network Layer -> Player Client

