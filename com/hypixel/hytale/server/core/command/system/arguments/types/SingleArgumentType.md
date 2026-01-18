---
description: Architectural reference for SingleArgumentType
---

# SingleArgumentType

**Package:** com.hypixel.hytale.server.core.command.system.arguments.types
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class SingleArgumentType<DataType> extends ArgumentType<DataType> {
```

## Architecture & Concepts
SingleArgumentType is a foundational abstract class within the server's command system framework. It serves as a specialized base for any command argument that consumes exactly one token (a single word or quoted string) from the command input.

Architecturally, this class employs the **Template Method** design pattern. The parent class, ArgumentType, defines the high-level contract for parsing an array of string tokens. SingleArgumentType implements this contract by handling the boilerplate logic of extracting the first token from the array (`input[0]`) and then delegating the actual type conversion to a new, simpler abstract method.

This design enforces a strong contract and simplifies development for the most common argument type. Implementors do not need to concern themselves with array bounds checking or the mechanics of token consumption; they can focus exclusively on the logic of parsing a single string into the target data type.

## Lifecycle & Ownership
- **Creation:** Concrete subclasses of SingleArgumentType are intended to be instantiated once at server startup. They are typically registered with a central command or argument type registry. They are stateless service objects, not per-request data objects.
- **Scope:** Application-wide. An instance of an argument type, such as an IntegerArgumentType, persists for the entire lifetime of the server.
- **Destruction:** Instances are garbage collected during server shutdown. There is no explicit destruction or cleanup method, as they hold no managed resources.

## Internal State & Concurrency
- **State:** **Stateless and Immutable.** The internal fields of this class (name, usage, examples) are set via the constructor and are not modified thereafter. All state related to a specific parsing operation is passed in via the method parameters, particularly the ParseResult object.
- **Thread Safety:** **Fully Thread-Safe.** Due to its stateless nature, a single instance of a SingleArgumentType subclass can be safely and concurrently invoked by multiple threads processing different commands without any need for synchronization or locks.

## API Surface
The primary contract is for extension by subclasses, not direct invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| parse(String, ParseResult) | DataType | Varies | **Abstract.** The core method subclasses must implement. Responsible for converting a single string token into the target DataType. Complexity depends entirely on the subclass implementation. |
| withOverriddenUsage(String, String...) | WrappedArgumentType | O(1) | A factory method that creates a lightweight wrapper around the current instance. This allows for customizing the usage message or examples without creating a new subclass. |

## Integration Patterns

### Standard Usage
Developers should **extend** this class to create new, reusable argument types. The primary responsibility is to implement the abstract `parse(String, ParseResult)` method.

```java
// A concrete implementation for parsing a boolean value.
public class BooleanArgumentType extends SingleArgumentType<Boolean> {

    public BooleanArgumentType() {
        super("boolean", "<true|false>", "true", "false");
    }

    @Override
    @Nullable
    public Boolean parse(String input, ParseResult parseResult) {
        if ("true".equalsIgnoreCase(input)) {
            return true;
        }
        if ("false".equalsIgnoreCase(input)) {
            return false;
        }
        
        // Report a failure back to the command system.
        parseResult.addError(Message.translation("command.error.invalid_boolean", input));
        return null;
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to use `new SingleArgumentType<>()`. The class is abstract and must be extended.
- **Multi-Token Logic:** Do not implement logic to parse multiple words inside a subclass of SingleArgumentType. This violates its core contract. For multi-token arguments, extend the base ArgumentType directly.
- **Stateful Implementations:** Subclasses must remain stateless. Storing parsing results or any other mutable state as instance fields will break thread safety and lead to severe, unpredictable bugs under load.

## Data Pipeline
SingleArgumentType is a critical component in the command processing pipeline. It is responsible for transforming raw string tokens into strongly-typed data that a command handler can safely use.

> Flow:
> Raw Command String -> Command Tokenizer -> Command Parser -> **SingleArgumentType Subclass** -> Typed Java Object -> Command Executor

