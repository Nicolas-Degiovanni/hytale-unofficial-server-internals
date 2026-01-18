---
description: Architectural reference for ProcessingBench
---

# ProcessingBench

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config.bench
**Type:** Configuration Model

## Definition
```java
// Signature
public class ProcessingBench extends Bench {
```

## Architecture & Concepts

The ProcessingBench class is a configuration model, not a live game-world entity. It serves as a static data template that defines the properties and behaviors of a processing-style interactive block, such as a furnace, kiln, or alloy forge. It extends the base Bench class, inheriting common properties for interactive blocks, and adds specialized fields for transformation-based crafting, including inputs, fuel, outputs, and processing sounds.

The cornerstone of this class's architecture is the static final **CODEC** field. This field leverages Hytale's declarative serialization system to map data from on-disk asset files (e.g., JSON) into this strongly-typed Java object. The BuilderCodec is responsible for parsing the asset, validating its fields against predefined rules (e.g., non-null, greater than or equal to 1), and constructing the final in-memory object.

A critical step in this process is the **afterDecode** hook defined within the CODEC. This function performs post-deserialization processing to derive runtime-optimized data. For example, it converts a string-based SoundEventId into an integer-based SoundEventIndex. This pre-computation avoids costly string lookups during gameplay, ensuring high performance in performance-critical game loops.

This object is intended to be read-only after its initial creation by the asset system. Game logic systems query instances of ProcessingBench to determine how to handle player interactions with a specific block type in the world.

## Lifecycle & Ownership

-   **Creation:** A ProcessingBench object is instantiated exclusively by the Hytale Codec system during the server's asset loading phase. The `AssetManager` discovers the corresponding asset file and uses the static CODEC definition to deserialize the data into a new instance. **WARNING:** Direct instantiation via `new ProcessingBench()` is a critical anti-pattern that bypasses the asset pipeline and will lead to uninitialized, non-functional objects.
-   **Scope:** Once loaded, an instance of ProcessingBench persists for the entire lifetime of the server. It is stored in the central asset registry and shared across all game systems. There is typically one unique ProcessingBench object for each distinct type of processing bench defined in the game's assets (e.g., one for a furnace, one for a kiln).
-   **Destruction:** The object is eligible for garbage collection only when the server shuts down and the asset registries are cleared.

## Internal State & Concurrency

-   **State:** The object's state is considered immutable after the `afterDecode` hook completes. All fields are populated once during asset loading. The `endSoundEventIndex` field is a prime example of derived, transient state that is calculated once at load time for runtime efficiency.
-   **Thread Safety:** This class is **thread-safe for reads**. As a shared, effectively immutable configuration object, multiple threads (e.g., world simulation threads, network threads) can safely access its getter methods without synchronization. **WARNING:** Any external attempt to mutate the state of a loaded ProcessingBench object at runtime is not thread-safe and will cause unpredictable behavior and race conditions across the server.

## API Surface

The public API consists primarily of getters. The following methods contain important business logic or provide access to runtime-optimized data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getInput(tierLevel) | ProcessingSlot[] | O(N) | Returns the valid input slots for the bench. The logic accounts for extra slots granted by a specific tier level, creating a new array. |
| getOutputSlotsCount(tierLevel) | int | O(1) | Returns the number of output slots, factoring in any extra slots provided by the specified tier level. |
| getEndSoundEventIndex() | int | O(1) | Returns the pre-calculated integer index for the processing completion sound. This is highly optimized for runtime use. |
| shouldAllowNoInputProcessing() | boolean | O(1) | Returns a flag indicating if the bench can process using only fuel, without any primary input items. |

## Integration Patterns

### Standard Usage

The canonical mechanism for accessing a ProcessingBench configuration is by retrieving it from a block type asset that a developer already has a reference to.

```java
// Assume 'block' is an instance of a Block in the game world
// The getConfig method retrieves the strongly-typed configuration asset
ProcessingBench benchConfig = block.getConfig(ProcessingBench.class);

if (benchConfig != null) {
    int outputCount = benchConfig.getOutputSlotsCount(currentTier);
    // Further game logic uses the retrieved configuration...
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new ProcessingBench()`. The object will be uninitialized and lack critical data derived by the codec, causing NullPointerExceptions and incorrect behavior.
-   **State Mutation:** Do not modify the fields of a ProcessingBench object retrieved from the asset system. This shared data is read by the entire server; runtime mutations will lead to severe, difficult-to-diagnose concurrency issues.
-   **String-Based Lookups in Loops:** Avoid calling `getEndSoundEventId()` repeatedly in performance-sensitive code. Use the pre-cached `getEndSoundEventIndex()` for efficient sound event dispatch.

## Data Pipeline

The ProcessingBench class is a product of the server's asset loading pipeline. Its data originates from a static file and is transformed into a read-only, in-memory representation for use by the game simulation.

> Flow:
> `asset.json` (On-Disk File) -> AssetManager (File Discovery) -> **ProcessingBench.CODEC** (Deserialization & Validation) -> **ProcessingBench Instance** (In-Memory Object) -> Game Systems (Read-Only Access)

