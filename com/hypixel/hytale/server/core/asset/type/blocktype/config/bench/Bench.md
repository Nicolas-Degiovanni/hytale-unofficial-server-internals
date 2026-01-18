---
description: Architectural reference for Bench
---

# Bench

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config.bench
**Type:** Configuration Model / Data Transfer Object

## Definition
```java
// Signature
public abstract class Bench implements NetworkSerializable<com.hypixel.hytale.protocol.Bench> {
```

## Architecture & Concepts
The Bench class is an abstract base model representing the server-side configuration for all interactive stations, such as crafting tables, furnaces, or anvils. Its primary role is to serve as a data container, deserialized from asset configuration files (e.g., JSON) during server initialization.

This class is deeply integrated with the engine's **Codec** system. The static fields CODEC and BASE_CODEC define a declarative schema for mapping raw data from asset files into a strongly-typed Java object. This pattern centralizes deserialization logic, validation, and post-processing, ensuring that all Bench instances are constructed consistently.

As an implementation of **NetworkSerializable**, a Bench object can be transformed into a network packet representation. This is a key part of the server-authoritative architecture: the server loads the complete configuration from disk, and then serializes a subset of this data to clients when needed. This ensures clients have the necessary information to display UIs and play sounds without having access to the server's full asset files.

A critical post-processing step occurs in the **afterDecode** hook. This function resolves human-readable string identifiers for sound events (e.g., localOpenSoundEventId) into high-performance integer indices (e.g., localOpenSoundEventIndex). This is a common engine optimization that combines the readability of configuration files with the runtime performance of integer-based lookups.

## Lifecycle & Ownership
- **Creation:** Bench instances are not meant to be instantiated directly using the *new* keyword. They are exclusively created by the asset loading system via the static CODEC field. The system reads a "Type" identifier from the data source to determine which concrete subclass of Bench to instantiate.
- **Scope:** An instance of a Bench configuration is loaded once when the server starts. It is a static, definitional asset that persists for the entire lifetime of the server process.
- **Destruction:** The object is garbage collected along with all other server assets during server shutdown. There is no manual destruction logic.

## Internal State & Concurrency
- **State:** The object is mutable during the deserialization process managed by the BuilderCodec. After the *afterDecode* hook completes, the object's state should be considered **effectively immutable**. The *transient* integer fields are a form of write-once cache populated during this phase.
- **Thread Safety:** This class is **not thread-safe** for mutation. It contains no internal locking. However, because its state is fixed after server initialization, it is safe for concurrent **read-only access** from multiple game threads.

**WARNING:** Modifying a Bench object at runtime after it has been loaded is a severe anti-pattern and will lead to unpredictable behavior and race conditions. Configuration data must remain static.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getTierLevel(int tier) | BenchTierLevel | O(1) | Retrieves the configuration for a specific tier level. Returns null if the tier is invalid. |
| get...SoundEventIndex() | int | O(1) | Returns the pre-calculated integer index for a sound event. Optimized for runtime use. |
| toPacket() | com.hypixel.hytale.protocol.Bench | O(N) | Serializes the object into a network packet. Complexity is proportional to the number of tier levels (N). |
| getRootInteraction() | RootInteraction | O(1) | **Deprecated.** Retrieves the interaction logic associated with this bench type from a static map. |

## Integration Patterns

### Standard Usage
Game logic should retrieve Bench configurations from a central asset manager or registry. Never instantiate or cache them manually.

```java
// Example: A block interaction handler
// Assume 'assetManager' is a service that holds all loaded game assets.

Bench benchConfig = assetManager.getBenchById("hytale:workbench");
if (benchConfig != null) {
    int openSoundIndex = benchConfig.getLocalOpenSoundEventIndex();
    audioSystem.playSound(player, openSoundIndex);
    // ... logic to open the bench UI for the player
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new CraftingBench()`. This bypasses the Codec system, leaving the object in a partially initialized state where critical fields, like sound indices, will be incorrect.
- **Runtime Modification:** Do not modify the state of a Bench object after it has been loaded. For example, do not change its tier levels or sound event IDs during gameplay.
- **Premature Access:** Do not access the integer-based indices (e.g., getLocalOpenSoundEventIndex) before the server's asset loading phase is fully complete. Doing so may result in a value of 0 if the sound asset map has not yet been populated.

## Data Pipeline
The flow of Bench data begins on disk and ends on the client, with the server acting as the authoritative intermediary.

> Flow:
> JSON Asset File -> Server Asset Loader -> **Bench.CODEC Deserialization** -> In-Memory Bench Instance -> **toPacket() Serialization** -> Network Packet -> Client Game Logic

