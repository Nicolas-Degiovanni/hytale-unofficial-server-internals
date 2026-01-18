---
description: Architectural reference for MacroCommandParameter
---

# MacroCommandParameter

**Package:** com.hypixel.hytale.builtin.commandmacro
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class MacroCommandParameter {
```

## Architecture & Concepts
The MacroCommandParameter class is a data-driven schema that defines the structure, type, and constraints of a single argument for a user-defined command macro. It is not a service or a manager, but rather a passive data container whose state is populated exclusively through the engine's serialization system.

Its primary architectural role is to act as a bridge between human-readable macro configuration files (e.g., JSON) and the server's internal command processing system. This is accomplished via the static **CODEC** field, a BuilderCodec that maps configuration keys like "Name" and "ArgType" to the object's internal fields during deserialization.

A key design feature is the nested **ArgumentTypeEnum**. This enum provides a simplified, user-friendly set of argument types (e.g., STRING, INTEGER, BLOCK_ID) in the configuration file. Internally, it maps these simple names to the engine's more complex and powerful ArgumentType implementations found in ArgTypes. This decouples the user-facing macro definition from the core command parsing engine, allowing the engine to evolve without breaking existing macro configurations.

In essence, a collection of MacroCommandParameter objects forms the signature of a command macro, which is then used by the MacroCommandSystem to construct a fully-featured, executable server command.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale serialization framework via the static **CODEC** field. This process is typically initiated by a higher-level system, such as a MacroCommandLoader, when parsing macro definition files from disk at server startup or during a live reload. Direct instantiation is an anti-pattern and will result in an uninitialized, non-functional object.

- **Scope:** The lifetime of a MacroCommandParameter instance is strictly bound to its parent MacroCommand. It persists in memory as part of the macro's definition for as long as that macro is registered with the server's command system.

- **Destruction:** The object is eligible for garbage collection when its parent MacroCommand is unregistered or the server shuts down. It has no explicit cleanup or destruction logic.

## Internal State & Concurrency
- **State:** A MacroCommandParameter object is **effectively immutable**. Its state is populated once during deserialization and is not intended to be modified thereafter. The public API only exposes getters, enforcing a read-only contract after initialization. It holds no runtime state and performs no caching.

- **Thread Safety:** The class is **thread-safe for reads**. As its internal state is fixed post-creation, multiple threads can safely access its properties simultaneously without locks or other synchronization primitives. The creation process itself is expected to occur in a single-threaded context during server initialization or a controlled reload procedure.

## API Surface
The public API consists solely of accessors for the deserialized configuration data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getName() | String | O(1) | Returns the programmatic name of the parameter. |
| getDescription() | String | O(1) | Returns the human-readable help text for the parameter. |
| getRequirement() | ParameterRequirement | O(1) | Returns the usage requirement (e.g., REQUIRED, OPTIONAL). |
| getArgumentType() | ArgumentTypeEnum | O(1) | Returns the enum representing the parameter's data type. |
| getDefaultValue() | String | O(1) | Returns the default value as a string, if applicable. |
| getDefaultValueDescription() | String | O(1) | Returns the help text for the default value. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly in Java code. Instead, they define its properties within a macro configuration file. The system then uses the deserialized object to build the command.

The conceptual usage by an internal system would look like this:

```java
// Pseudo-code for engine internals
MacroCommand command = loadMacroFromFile("my_macro.json");

for (MacroCommandParameter param : command.getParameters()) {
    // Use the parameter's definition to build a real command argument
    CommandArgument.Builder builder = new CommandArgument.Builder();
    builder.setName(param.getName());
    builder.setType(param.getArgumentType().getArgumentType()); // Maps enum to engine type
    
    if (param.getRequirement() == ParameterRequirement.OPTIONAL) {
        builder.setOptional(true);
    }
    
    // ... add builder to the final command structure
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new MacroCommandParameter()`. The resulting object will have null fields and will cause NullPointerExceptions in the command system. All creation must be handled by the framework's deserializer using the static CODEC.

- **State Modification:** Do not use reflection to modify the fields of a MacroCommandParameter after it has been created. The command system relies on this data being immutable; runtime changes can lead to undefined behavior and command parsing failures.

## Data Pipeline
The MacroCommandParameter is a product of a data-in pipeline. Its state is defined entirely by external configuration files.

> Flow:
> Macro Definition File (JSON) -> Engine File Loader -> **BuilderCodec Deserializer** -> **MacroCommandParameter Instance** -> MacroCommandSystem -> Server Command Registry

