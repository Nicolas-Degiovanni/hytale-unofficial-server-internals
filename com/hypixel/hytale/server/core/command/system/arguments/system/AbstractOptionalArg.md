---
description: Architectural reference for AbstractOptionalArg
---

# AbstractOptionalArg

**Package:** com.hypixel.hytale.server.core.command.system.arguments.system
**Type:** Component

## Definition
```java
// Signature
public abstract class AbstractOptionalArg<Arg extends Argument<Arg, DataType>, DataType> extends Argument<Arg, DataType> {
```

## Architecture & Concepts

AbstractOptionalArg is the foundational component for all optional arguments within the server's command processing system. It extends the base Argument class to introduce a powerful dependency and validation engine, enabling the creation of complex and context-aware command structures.

Its primary architectural role is to serve as a stateful builder during command definition and later as a stateless validation rule during command parsing. A command, represented by an AbstractCommand, is composed of a collection of these argument objects. AbstractOptionalArg allows developers to define intricate relationships between these arguments, such as making one argument required only when another is present, or making an argument available only to users with a specific permission.

This class embodies the **Builder Pattern** for its fluent configuration API and acts as a key participant in a **Composite Pattern**, where an AbstractCommand is the composite entity composed of various Argument leaves.

## Lifecycle & Ownership

The lifecycle of an AbstractOptionalArg is strictly controlled and tied to its parent command. Any deviation from this lifecycle will result in an IllegalStateException.

-   **Creation:** An AbstractOptionalArg is instantiated exclusively within the constructor or initialization block of a concrete AbstractCommand implementation. It is never created in isolation. The parent command passes a reference to itself (`this`) during construction, establishing a permanent ownership link.

-   **Scope:** The object persists for the entire lifetime of its parent AbstractCommand. Its scope is identical to that of the command's registration within the central CommandManager.

-   **Destruction:** The object is marked for garbage collection when its parent AbstractCommand is unregistered from the CommandManager and no other references exist. There are no explicit cleanup or `destroy` methods.

## Internal State & Concurrency

-   **State:** The internal state is highly **mutable** during the configuration phase—that is, before its parent command is registered. Fluent API calls like `requiredIf` and `addAliases` actively modify internal Sets that define its validation logic. Once the parent command is registered, the state becomes **effectively immutable**. The system enforces this immutability by throwing an IllegalStateException if any configuration method is called post-registration.

-   **Thread Safety:** This class is **not thread-safe** and is designed for single-threaded access during its configuration phase. All setup must occur synchronously during server bootstrap. The validation method, `verifyArgumentDependencies`, is safe to be called from multiple threads (e.g., per-player command execution threads) *only after* the command has been registered, as the internal state is guaranteed to be static at that point.

## API Surface

The public API is designed as a fluent interface for building dependency rules.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addAliases(String... newAliases) | Arg | O(N) | Adds one or more alternative names for the argument. Throws IllegalStateException if called after command registration. |
| requiredIf(AbstractOptionalArg... deps) | Arg | O(N) | This argument becomes mandatory if any of the dependent arguments are provided in the command context. |
| requiredIfAbsent(AbstractOptionalArg... deps) | Arg | O(N) | This argument becomes mandatory if any of the dependent arguments are *not* provided in the command context. |
| availableOnlyIfAll(AbstractOptionalArg... deps) | Arg | O(N) | This argument is only considered valid if all dependent arguments are provided. |
| availableOnlyIfAllAbsent(AbstractOptionalArg... deps) | Arg | O(N) | This argument is only considered valid if all dependent arguments are absent. |

| setPermission(String permission) | Arg | O(1) | Assigns a permission node required to use this specific argument. |
| verifyArgumentDependencies(context, result) | boolean | O(D) | Executes all configured dependency rules against the current command context. Populates the ParseResult on failure. D is the total number of dependencies. |
| hasPermission(CommandSender sender) | boolean | O(1) | Checks if the sender possesses the permission node associated with this argument. |

## Integration Patterns

### Standard Usage

Correct usage involves defining and configuring arguments within the constructor of an AbstractCommand, before it is registered. The fluent API is used to chain dependency rules.

```java
// Inside the constructor of a concrete AbstractCommand
// Define two optional arguments: a target player and a reason.
OptionalPlayerArgument targetArg = new OptionalPlayerArgument(this, "target", "The target player.");
OptionalStringArgument reasonArg = new OptionalStringArgument(this, "reason", "The reason for the action.");

// Make the 'reason' argument required ONLY if a 'target' is specified.
reasonArg.requiredIf(targetArg);

// Add the configured arguments to the command.
this.addArgument(targetArg);
this.addArgument(reasonArg);
```

### Anti-Patterns (Do NOT do this)

-   **Post-Registration Modification:** Attempting to call any configuration method (`addAliases`, `requiredIf`, `setPermission`, etc.) after the parent command has been registered with the CommandManager will fail. The object is locked to prevent runtime inconsistencies.

    ```java
    // BAD: This code will throw an IllegalStateException
    MyCommand command = new MyCommand();
    commandManager.register(command);
    command.getReasonArgument().setPermission("my.new.permission"); // Fails
    ```

-   **Circular Dependencies:** The framework does not explicitly detect circular dependencies. Defining rules such as `argA.requiredIf(argB)` and `argB.requiredIf(argA)` can create logically impossible validation states that will always fail parsing.

-   **Contradictory Rules:** The system prevents adding the same dependency to mutually exclusive rule sets (e.g., `requiredIf` and `requiredIfAbsent`), but complex chains of dependencies can still lead to contradictions that must be resolved by the developer.

## Data Pipeline

AbstractOptionalArg is a critical gatekeeper in the command parsing and validation pipeline. It does not transform data but rather validates the structure of the incoming command based on its pre-configured rules.

> Flow:
> Raw Command String -> Command Parser -> **AbstractOptionalArg.verifyArgumentDependencies** -> Command Executor -> Action Result

