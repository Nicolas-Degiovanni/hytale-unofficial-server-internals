---
description: Architectural reference for MacroCommandBuilder
---

# MacroCommandBuilder

**Package:** com.hypixel.hytale.builtin.commandmacro
**Type:** Data Transfer Object / Builder

## Definition
```java
// Signature
public class MacroCommandBuilder implements JsonAssetWithMap<String, DefaultAssetMap<String, MacroCommandBuilder>> {
```

## Architecture & Concepts
The MacroCommandBuilder serves as a deserialization target and intermediate representation for command macros defined in external JSON asset files. It is not an executable command itself, but rather a blueprint used by the engine to construct and register a functional command at runtime.

Its primary architectural role is to decouple the on-disk configuration of a command (its name, aliases, parameters, and sequence of actions) from its in-memory, executable representation, which is the MacroCommandBase.

The static field **CODEC** is the most critical component of this class. It leverages the engine's powerful `AssetBuilderCodec` system to define a strict contract for how a JSON object is parsed and mapped to the fields of a MacroCommandBuilder instance. This mechanism enables a fully data-driven approach to creating new in-game commands without requiring new Java code for each one.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale AssetStore during the asset loading phase. The `AssetBuilderCodec` invokes the private constructor when it encounters a corresponding JSON asset file that defines a command macro. **Warning:** Manual instantiation by developers is an unsupported and critical anti-pattern.

- **Scope:** Transient and short-lived. An instance of MacroCommandBuilder exists only for the duration of its asset being processed. Its lifecycle is bound to the asset loading and command registration sequence.

- **Destruction:** After the static `createAndRegisterCommand` method is called, the builder's data has been used to construct a persistent MacroCommandBase object. The MacroCommandBuilder instance itself is no longer referenced by the core engine and becomes eligible for garbage collection.

## Internal State & Concurrency
- **State:** The object is highly mutable during the deserialization process, as the `AssetBuilderCodec` populates its fields sequentially. Once fully loaded and passed to the registration logic, it should be treated as an immutable data container.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and consumed on a single thread, typically the main engine thread or a dedicated asset loading thread. Passing a MacroCommandBuilder instance between threads or attempting to modify it concurrently will result in race conditions and undefined behavior. All command registration must be synchronized.

## API Surface
The public API is minimal, as the class is primarily intended for internal system use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| createAndRegisterCommand(builder) | static CommandRegistration | O(log N) | Consumes a builder to construct and register a MacroCommandBase. Returns null if the builder is invalid. |
| getName() | String | O(1) | Returns the primary name of the command defined in the asset. |
| getId() | String | O(1) | Returns the unique asset identifier for this command macro. |

## Integration Patterns

### Standard Usage
Direct interaction with this class is rare. A developer's primary interaction is by defining a JSON asset. The system then uses the MacroCommandBuilder internally to load that asset and register the command. The conceptual flow within the engine is as follows.

```java
// This code is executed internally by the AssetStore and MacroCommandPlugin

// 1. The AssetStore deserializes the JSON into a builder instance.
MacroCommandBuilder loadedBuilder = assetStore.load("my_macro.json");

// 2. The plugin consumes the builder to register the final command.
if (loadedBuilder != null) {
    MacroCommandBuilder.createAndRegisterCommand(loadedBuilder);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new MacroCommandBuilder()`. The object's state is meant to be populated exclusively by the asset codec from a data source. Manual creation will result in a partially initialized, invalid object.

- **State Mutation:** Do not modify a builder's fields after it has been loaded. The state is a direct reflection of the on-disk asset and should not be altered at runtime.

- **Reusing Instances:** Do not hold a reference to a MacroCommandBuilder after it has been used to register a command. Its lifecycle is intended to be transient.

## Data Pipeline
The MacroCommandBuilder is a key link in the chain that transforms a static asset file into a live, executable command within the server's command registry.

> Flow:
> JSON Asset File -> AssetStore Loader -> **MacroCommandBuilder** (Deserialization Target) -> `createAndRegisterCommand` -> MacroCommandBase (Live Command) -> CommandRegistry

