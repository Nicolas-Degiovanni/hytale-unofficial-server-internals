---
description: Architectural reference for RootInteraction
---

# RootInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config
**Type:** Configuration Asset

## Definition
```java
// Signature
public class RootInteraction
   implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, RootInteraction>>,
   NetworkSerializable<com.hypixel.hytale.protocol.RootInteraction> {
```

## Architecture & Concepts

The RootInteraction class is the primary entry point for defining and executing complex action sequences within the game engine, such as player attacks, item usage, or NPC behaviors. It is not a service or a manager, but rather a data-driven configuration asset loaded from JSON files.

Architecturally, it serves as the top-level container for a chain of one or more child Interaction assets. Its primary responsibility is to aggregate shared properties—such as cooldowns, activation rules, and game mode-specific settings—and to compile the referenced child Interactions into an optimized, executable sequence of Operations.

This compilation step, performed by the build method, is a critical design feature. It transforms a declarative list of string identifiers (interactionIds) into a direct array of executable Operation objects. This decouples the configuration data from the runtime execution logic and provides a significant performance benefit by pre-resolving all dependencies after the asset is loaded.

Instances of RootInteraction are managed exclusively by the global AssetStore, ensuring that for any given interaction ID (e.g., "hytale:player_primary_attack"), there is a single, shared instance for the entire server.

## Lifecycle & Ownership

-   **Creation:** Instances are created exclusively by the Hytale AssetStore during server initialization or on-demand asset loading. The static CODEC field dictates the entire deserialization process from JSON source files. Direct instantiation via the constructor is strictly forbidden.

-   **Scope:** Application-scoped. Once an instance is loaded into the AssetStore, it persists for the entire server session. It is effectively a singleton within the context of its unique asset ID.

-   **Destruction:** Instances are destroyed only when the server shuts down and the AssetRegistry is cleared. There is no mechanism for manual destruction or unloading of individual RootInteraction assets.

## Internal State & Concurrency

-   **State:** The object's state is **Mutable** only during a brief, well-defined "linking" phase immediately after deserialization. The build method populates the internal operations array and calculates the needsRemoteSync flag. After this one-time compilation, the object must be treated as **Effectively Immutable**. Any runtime modification to its state is unsupported and will lead to severe concurrency issues and undefined behavior.

-   **Thread Safety:** The class is **not thread-safe** for mutation. The build method is an internal system process and must not be invoked concurrently. However, once the build phase is complete, the object is **safe for concurrent reads** from any number of game logic threads. This is the expected and standard usage pattern.

    **Warning:** The static getAssetStore method contains a lazy initializer. While typically safe due to being called during single-threaded server startup, invoking it for the first time from multiple threads simultaneously could theoretically create a race condition.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Retrieves the global asset manager for all RootInteraction instances. |
| build() | void | O(N) | **Internal Use Only.** Compiles child interaction IDs into an executable operation array. N is the number of child interactions. |
| toPacket() | com.hypixel.hytale.protocol.RootInteraction | O(N) | Serializes the object into a network packet for synchronization with clients. |
| getOperation(int index) | Operation | O(1) | Retrieves a pre-compiled operation from the execution chain. Returns null if the index is out of bounds. |
| getId() | String | O(1) | Returns the unique string identifier for this asset. |
| getCooldown() | InteractionCooldown | O(1) | Returns the configured cooldown object, or null if none exists. |
| getRules() | InteractionRules | O(1) | Returns the set of rules governing the execution of this interaction chain. |

## Integration Patterns

### Standard Usage

A system should always retrieve a RootInteraction instance from the central AssetStore via its static accessor. The retrieved object is then used to access the pre-compiled operations for execution.

```java
// Correctly retrieve a shared, pre-compiled RootInteraction asset
IndexedLookupTableAssetMap<String, RootInteraction> assetMap = RootInteraction.getAssetMap();
RootInteraction playerAttack = assetMap.getAsset("hytale:player_primary_attack");

if (playerAttack != null) {
    // Execute the first operation in the chain
    Operation firstOp = playerAttack.getOperation(0);
    if (firstOp != null) {
        // ... game logic to execute the operation
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new RootInteraction()`. This bypasses the AssetStore and the critical build process, resulting in a non-functional object with a null operations array. All instances must be defined in JSON and loaded by the engine.

-   **Manual Re-compilation:** Do not call the build method on an already-loaded RootInteraction. This method is designed for internal, post-deserialization setup. Calling it at runtime can lead to unpredictable behavior, especially if other threads are accessing the object.

-   **State Modification:** Do not modify any fields on a retrieved RootInteraction instance. These objects are shared across the entire server. Modifying a cooldown or rule set at runtime will affect all game logic and is not thread-safe.

## Data Pipeline

The RootInteraction class sits at the center of two primary data flows: asset loading and network synchronization.

**Asset Loading & Compilation Pipeline:**
> Flow:
> JSON Asset File -> AssetStore Deserializer (using CODEC) -> Raw **RootInteraction** Instance -> Internal build() call -> Compiled **RootInteraction** with `operations` array -> Game Logic Systems

**Network Synchronization Pipeline:**
> Flow:
> Server-side **RootInteraction** -> toPacket() method -> Network Packet -> Client -> Deserialized into client-side representation for prediction and effects.

