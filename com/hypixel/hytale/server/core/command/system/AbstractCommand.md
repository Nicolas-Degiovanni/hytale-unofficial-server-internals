---
description: Architectural reference for AbstractCommand
---

# AbstractCommand

**Package:** com.hypixel.hytale.server.core.command.system
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class AbstractCommand {
```

## Architecture & Concepts
The **AbstractCommand** class is the foundational component of the Hytale server's command processing framework. It is not a concrete command but rather a blueprint from which all server commands, both system-level and plugin-provided, must inherit. Its design promotes a declarative, hierarchical, and type-safe approach to command creation.

The core architectural pattern is a **Composite Tree**. An **AbstractCommand** instance can contain a collection of child **subCommands** and **variantCommands**, forming a tree structure. This allows for complex, multi-level command syntaxes like */zone modify setflag pvp deny*.

Key architectural responsibilities include:
- **Command Definition:** Provides a fluent API (e.g., **withRequiredArg**, **withOptionalArg**) for developers to declare a command's name, aliases, permissions, and argument signature.
- **Parsing & Validation:** Contains the complete logic for parsing a raw input string, validating it against the defined signature, and resolving required, optional, flag, and default arguments.
- **Lifecycle Management:** Enforces a strict registration lifecycle. A command is mutable during its initial configuration but becomes locked after being registered with the **CommandManager**, preventing unsafe runtime modifications.
- **Execution Orchestration:** Acts as the entry point for a command call. It processes the input, checks permissions, and, upon successful validation, populates a **CommandContext** object which is then passed to the concrete implementation's **execute** method.
- **Variant Support:** Uniquely supports command "overloads" through **variantCommands**. This allows a single command name (e.g., */teleport*) to have different behaviors and required arguments based on the number of parameters provided by the user.

This class serves as the bridge between the raw text input from a **CommandSender** and the structured, type-safe execution logic within a concrete command implementation.

### Lifecycle & Ownership
The lifecycle of an **AbstractCommand** instance is rigidly controlled by the command registration process to ensure system stability.

- **Creation:** Concrete subclasses of **AbstractCommand** are instantiated by server systems or plugins during their initialization phase. The constructor is used to define the command's signature by chaining calls to argument registration methods like **withRequiredArg**.
- **Scope:** An **AbstractCommand** object persists for the lifetime of its owner. If registered by a plugin, it lives until the plugin is disabled. If it is a core server command, it persists for the entire server session. The **CommandOwner** (typically a **PluginBase** or the global **CommandManager**) is responsible for holding the root reference.
- **Destruction:** There is no explicit destruction method. Instances are eligible for garbage collection when their owning plugin is unloaded or the server shuts down, and all references within the **CommandManager** are cleared.

**WARNING:** Modifying a command's structure (e.g., adding subcommands or arguments) after it has been registered is strictly forbidden and will result in an **IllegalStateException**. All definition must occur before the **completeRegistration** method is invoked by the framework.

## Internal State & Concurrency
- **State:** The internal state of **AbstractCommand** is highly mutable during its configuration phase. Collections for subcommands, aliases, and arguments are populated before registration. After the **completeRegistration** method is called, the command's definition is effectively immutable. The **hasBeenRegistered** flag acts as a guard against post-registration modifications.

- **Thread Safety:** This class is **not thread-safe** for configuration but is safe for concurrent execution.
    - **Configuration:** All registration methods (**addSubCommand**, **withRequiredArg**, etc.) are unsynchronized and must be called from a single thread during server or plugin initialization.
    - **Execution:** The primary execution path, **acceptCall**, is re-entrant. It does not modify the command's internal definition. It creates transient objects (**CommandContext**, **ParseResult**) for each invocation, making it safe to be called by multiple threads simultaneously, which is typical in a server environment with many players executing commands.

## API Surface
The public API is divided into two categories: configuration methods for defining the command and execution methods for processing it.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(CommandContext) | abstract CompletableFuture | Implementation-Defined | The core logic of the command, implemented by subclasses. |
| addSubCommand(AbstractCommand) | void | O(N) | Registers a child command, building the command tree. N is number of aliases. |
| addUsageVariant(AbstractCommand) | void | O(1) | Registers a command overload based on the number of required arguments. |
| withRequiredArg(...) | RequiredArg | O(1) | Declares a mandatory argument for the command. |
| withOptionalArg(...) | OptionalArg | O(1) | Declares an optional, named argument. |
| acceptCall(...) | CompletableFuture | O(T + A) | The main entry point for parsing and executing a command call. T is tokens, A is arguments. |
| hasPermission(CommandSender) | boolean | O(P) | Recursively checks if the sender has permission for this command and its parents. P is parent depth. |
| getUsageString(CommandSender) | Message | O(A + S) | Generates a detailed, multi-line help message. A is arguments, S is subcommands. |

## Integration Patterns

### Standard Usage
A developer defines a command by extending **AbstractCommand**, declaring its arguments in the constructor, and implementing the **execute** method. The instance is then registered with the central **CommandManager**.

```java
// 1. Define the command
public class HealCommand extends AbstractCommand {
    private final RequiredArg<Player> playerArg;

    public HealCommand() {
        super("heal", "Heals a target player to full health.");
        this.playerArg = withRequiredArg("player", "The player to heal.", ArgumentTypes.PLAYER);
        requirePermission("myplugin.command.heal");
    }

    @Override
    protected CompletableFuture<Void> execute(@Nonnull CommandContext context) {
        Player target = context.get(this.playerArg);
        target.setHealth(target.getMaxHealth());
        context.sender().sendMessage(Message.raw("Healed " + target.getName()));
        return CompletableFuture.completedFuture(null);
    }
}

// 2. Register the command during plugin initialization
CommandManager commandManager = server.getCommandManager();
commandManager.registerCommand(new HealCommand());
```

### Anti-Patterns (Do NOT do this)
- **Post-Registration Modification:** Calling methods like **addSubCommand** or **withRequiredArg** after the command has been registered with the **CommandManager**. This will throw an **IllegalStateException**.
- **Argument or Subcommand Re-use:** Attempting to register the same **AbstractCommand** or **Argument** instance as a child of multiple different parent commands. The framework explicitly prevents this to maintain a clean ownership hierarchy.
- **Ignoring the CommandContext:** Directly accessing global state or player managers within the **execute** method without using the provided **CommandSender** and parsed arguments from the **CommandContext**. This breaks encapsulation and makes commands harder to test.

## Data Pipeline
The flow of data from user input to command execution is a multi-stage pipeline orchestrated by **AbstractCommand**.

> Flow:
> Raw String Input -> **CommandManager** -> Tokenization -> **ParserContext** -> **AbstractCommand.acceptCall** -> Permission Check -> Argument Parsing -> **CommandContext** -> **AbstractCommand.execute** -> Asynchronous Result (CompletableFuture)

