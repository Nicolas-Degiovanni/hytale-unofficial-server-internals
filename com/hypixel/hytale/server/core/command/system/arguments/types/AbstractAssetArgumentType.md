---
description: Architectural reference for AbstractAssetArgumentType
---

# AbstractAssetArgumentType

**Package:** com.hypixel.hytale.server.core.command.system.arguments.types
**Type:** Framework Component

## Definition
```java
// Signature
public abstract class AbstractAssetArgumentType<DataType extends JsonAssetWithMap<AssetKeyType, M>, M extends AssetMap<AssetKeyType, DataType>, AssetKeyType>
   extends SingleArgumentType<DataType> {
```

## Architecture & Concepts
AbstractAssetArgumentType is a foundational component within the server's command parsing framework. It serves as a generic, reusable bridge between raw string input from a command and strongly-typed game asset objects. Its primary architectural role is to standardize the process of resolving an asset identifier (like *hytale:stone*) into a live game data object (a Block instance).

This class decouples command logic from the specifics of asset loading and storage. By extending this base class, developers can easily create new command argument types for any asset managed by the AssetRegistry—such as items, blocks, prefabs, or sounds—without rewriting complex lookup, error handling, or user-facing feedback logic.

Key architectural features include:
*   **Centralized Lookup:** It funnels all asset resolution requests through the global AssetRegistry, ensuring that commands always operate on the canonical, authoritative set of game data.
*   **User-Friendly Error Handling:** It automatically generates "did you mean" suggestions for failed lookups using fuzzy string matching, providing a consistent and helpful user experience across all commands that use asset arguments.
*   **Type Safety:** The use of generics ensures that a command expecting a Block cannot be accidentally provided with an Item, enforcing type safety at the command definition level.

## Lifecycle & Ownership
- **Creation:** Instances of concrete subclasses (e.g., BlockArgumentType, ItemArgumentType) are instantiated once during server bootstrap. They are then registered with the central CommandSystem to become part of the static command graph.
- **Scope:** An instance persists for the entire server session. It is a long-lived, stateless service object.
- **Destruction:** Instances are discarded and garbage collected only when the server shuts down and the CommandSystem itself is dismantled.

## Internal State & Concurrency
- **State:** This class is effectively stateless. The internal field dataTypeClass is immutable after construction. It holds no caches or mutable data; all stateful operations, such as asset lookups, are delegated to the AssetRegistry.
- **Thread Safety:** The class is thread-safe for its intended use case. The primary method, parse, performs read-only operations against the global AssetRegistry. Assuming the AssetRegistry is thread-safe for concurrent reads, this component can be safely used by multiple command execution threads simultaneously without external locking.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| parse(input, parseResult) | DataType | O(N) | Attempts to resolve a string input into a game asset. On failure, it populates the ParseResult with detailed error messages and suggestions. The complexity is dominated by the fuzzy search on failure, where N is the total number of assets of the target type. |
| getAssetKey(input) | AssetKeyType | - | **Abstract.** Subclasses must implement this to convert the raw string input into a valid key for the AssetMap. |
| getAssetMap() | M | O(1) | Retrieves the corresponding asset map from the global AssetRegistry. Throws an exception if the asset store is not registered. |

## Integration Patterns

### Standard Usage
The primary pattern is to extend this class to create a new, specific argument type for a given game asset. The developer only needs to provide the logic for converting a string to an AssetKeyType.

```java
// 1. Define a concrete implementation for a "Block" asset.
// This class translates a string like "hytale:stone" into a HytaleResourceKey.
public class BlockArgumentType extends AbstractAssetArgumentType<Block, BlockMap, HytaleResourceKey> {
    public BlockArgumentType() {
        // Register with the name "block", the type Block.class, and a usage hint.
        super("block", Block.class, "<block_id>");
    }

    @Override
    @Nullable
    public HytaleResourceKey getAssetKey(@Nonnull String input) {
        // The core logic: convert the string to a resource key.
        return HytaleResourceKey.fromString(input);
    }
}

// 2. During server startup, register the new type with the command system.
// This makes the "block" argument type available to all command builders.
CommandSystem.getArgumentRegistry().register(new BlockArgumentType());
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Subclasses should not contain mutable state. Argument types are shared across all commands and threads, and storing state can lead to severe concurrency bugs.
- **Expensive Key Conversion:** The getAssetKey method is called frequently during command parsing. Implementing it with a slow or resource-intensive operation (e.g., a database query, complex computation) will create a performance bottleneck for the entire command system.
- **Circumventing AssetRegistry:** Do not attempt to load assets or manage your own asset cache within a subclass. Always delegate lookups to the AssetRegistry via the provided getAssetMap method to ensure data consistency.

## Data Pipeline
This component acts as a validation and transformation step within the broader command processing pipeline.

> Flow:
> Raw Command String -> Command Parser Tokenizer -> **AbstractAssetArgumentType.parse()** -> AssetRegistry Lookup -> Typed Asset Object -> Command Executor Logic

