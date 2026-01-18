---
description: Architectural reference for AssetArgumentType
---

# AssetArgumentType

**Package:** com.hypixel.hytale.server.core.command.system.arguments.types
**Type:** Transient

## Definition
```java
// Signature
public class AssetArgumentType<DataType extends JsonAssetWithMap<String, M>, M extends AssetMap<String, DataType>>
   extends AbstractAssetArgumentType<DataType, M, String> {
```

## Architecture & Concepts
The AssetArgumentType is a specialized parser within the server's Command System framework. Its primary function is to interpret a raw string argument from a command as a direct key for an asset, such as an item, block, or entity definition.

This class acts as a bridge between user input and the engine's AssetStore. It validates that a given string corresponds to a loaded asset of a specific type, transforming a simple string like *hytale:stone* into a strongly-typed, engine-aware object reference.

Architecturally, it is a concrete implementation of the Strategy Pattern, where the parent AbstractAssetArgumentType defines the contract for parsing asset-based arguments. This specific implementation embodies the simplest strategy: the input string is treated as the literal asset key without any transformation.

## Lifecycle & Ownership
- **Creation:** Instantiated once during server initialization or plugin loading when a command is being defined and registered with the CommandDispatcher. It is constructed as part of a command's metadata.
- **Scope:** The object's lifecycle is bound to the command it helps define. It persists in memory as long as its associated command is registered and available for execution.
- **Destruction:** De-referenced and eligible for garbage collection when its parent command is unregistered, which typically occurs during a server shutdown or a hot-reload of game modules.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state, consisting of the argument name, target asset class, and usage string, is set exclusively via the constructor and cannot be modified thereafter. This class holds no runtime state or cached data.
- **Thread Safety:** **Fully Thread-Safe**. Due to its immutable and stateless nature, a single instance can be safely and concurrently invoked by multiple threads processing commands without requiring any synchronization or locking mechanisms. This is a critical design feature for a high-performance, multi-threaded server environment.

## API Surface
The primary public contract is the constructor for instantiation and the inherited parsing logic which uses the `getAssetKey` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetKey(String input) | String | O(1) | Returns the asset key to be used for lookup. This implementation performs an identity transformation, returning the input string unmodified. |

## Integration Patterns

### Standard Usage
AssetArgumentType is intended to be used declaratively when building a command that requires a player to specify a game asset by its unique identifier.

```java
// Example: Defining a /give command argument
// The command system will use this instance to parse the "item" argument.
CommandBuilder.create("give")
    .withArgument("player", new PlayerArgumentType())
    .withArgument("item", new AssetArgumentType<>("item", Item.class, "<item_id>"))
    .withArgument("count", new IntegerArgumentType("count"))
    .executes(context -> {
        // The context would provide the parsed, validated Item object
        Item requestedItem = context.getArgument("item", Item.class);
        // ... command logic
    });
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation for Parsing:** Do not create a new AssetArgumentType just to parse a string. It is designed to be registered with the command system, which handles the parsing lifecycle.
- **Complex Key Logic:** Do not subclass this type to add simple prefixing or other minor string manipulations. For complex key derivation, extend the parent AbstractAssetArgumentType directly and provide a more sophisticated `getAssetKey` implementation.

## Data Pipeline
This component sits early in the command processing pipeline, responsible for validation and type conversion. An invalid asset key will terminate the pipeline for that command, returning an error to the user.

> Flow:
> Raw Command String -> CommandDispatcher -> **AssetArgumentType** -> AssetStore Lookup -> Typed Asset Object -> Command Executor

