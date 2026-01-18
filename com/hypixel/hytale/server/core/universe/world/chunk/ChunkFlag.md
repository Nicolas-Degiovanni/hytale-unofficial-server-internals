---
description: Architectural reference for ChunkFlag, a bitmask-based state definition for world chunks.
---

# ChunkFlag

**Package:** com.hypixel.hytale.server.core.universe.world.chunk
**Type:** Type Definition / Utility

## Definition
```java
// Signature
public enum ChunkFlag implements Flag {
```

## Architecture & Concepts

ChunkFlag is a type-safe enumeration that defines the possible states and attributes of a World Chunk. It is a foundational component of the server's chunk lifecycle management system.

The core architectural pattern employed here is **bitwise state management**. Instead of using a collection of boolean fields to track a chunk's state (e.g., isInitialized, isTicking), the system uses a single integer as a bitfield. Each ChunkFlag constant represents a specific, non-overlapping bit within that integer.

This design provides significant performance and memory advantages:
*   **Efficiency:** Checking or modifying multiple states can be achieved with a few cheap, atomic CPU instructions (AND, OR, XOR). This is critical in performance-sensitive systems like world simulation.
*   **Memory:** A single integer can store up to 32 distinct boolean flags, drastically reducing the memory footprint of each chunk object compared to individual boolean fields.
*   **Atomicity:** Modifying the state integer is often an atomic operation, simplifying concurrency control in the chunk management system.

By implementing the common Flag interface, ChunkFlag participates in a standardized, engine-wide pattern for bitmasking, ensuring consistency with other flag-based systems.

### Lifecycle & Ownership
*   **Creation:** ChunkFlag constants are instantiated by the Java Virtual Machine during class loading. They are compile-time constants, not runtime objects in the traditional sense.
*   **Scope:** The constants are static and exist for the entire lifetime of the server application. They are effectively global, immutable singletons.
*   **Destruction:** The enum and its constants are unloaded only when the JVM shuts down. There is no concept of manual lifecycle management for this type.

## Internal State & Concurrency
*   **State:** Deeply and fundamentally immutable. The state of each enum constant, including its name, ordinal, and pre-calculated bitmask, is fixed at compile time and cannot be altered.
*   **Thread Safety:** ChunkFlag is inherently thread-safe due to its immutability. Its constants can be safely accessed and read from any thread without requiring locks or other synchronization primitives. The responsibility for thread-safe *modification* of the state integer that *uses* these flags lies with the owning class, such as the Chunk itself.

## API Surface

The primary API is the set of defined constants. The methods provide the necessary machinery for the bitwise architecture.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| START_INIT | ChunkFlag | O(1) | Constant representing the initial state when a chunk begins its loading or generation process. |
| INIT | ChunkFlag | O(1) | Constant representing a fully initialized chunk whose data is ready for use by the game simulation. |
| NEWLY_GENERATED | ChunkFlag | O(1) | Constant indicating the chunk was created via procedural generation, not loaded from storage. |
| ON_DISK | ChunkFlag | O(1) | Constant indicating the chunk has a serialized representation on disk and can be unloaded. |
| TICKING | ChunkFlag | O(1) | Constant indicating the chunk is active and should be processed by the main world simulation tick. |
| mask() | int | O(1) | **Critical API.** Returns the pre-calculated integer bitmask for the flag, used in all bitwise operations. |

## Integration Patterns

ChunkFlag is not used in isolation. It is designed to be integrated with a state holder class, typically the core Chunk object, which maintains an integer or long field for its flags. Systems like the ChunkManager, WorldGenerator, and WorldSerializer then interact with this integer field using the masks provided by ChunkFlag.

### Standard Usage

The correct usage pattern involves retrieving the mask from the desired flag and applying bitwise operators to a state integer.

```java
// Assume 'chunk' is an object with a 'flags' integer field.
// int chunk.flags;

// To set a flag (e.g., mark the chunk as initialized)
chunk.flags |= ChunkFlag.INIT.mask();

// To set multiple flags at once
chunk.flags |= (ChunkFlag.TICKING.mask() | ChunkFlag.ON_DISK.mask());

// To check if a flag is set
boolean isInitialized = (chunk.flags & ChunkFlag.INIT.mask()) != 0;

// To remove a flag (e.g., stop the chunk from ticking)
chunk.flags &= ~ChunkFlag.TICKING.mask();
```

### Anti-Patterns (Do NOT do this)
*   **Ordinal Comparison:** Never use the enum ordinal for logic (e.g., `if (flag.ordinal() == 1)`). The ordinal is an implementation detail of the mask calculation and is not guaranteed to be stable. The bitmask is the public contract.
*   **String-Based Logic:** Avoid using `flag.name()` in performance-critical paths for state checks. Bitwise integer operations are orders of magnitude faster than string comparisons.
*   **Direct Mask Calculation:** Do not replicate the mask calculation logic (e.g., `1 << flag.ordinal()`). Always use the provided `mask()` method to respect the API contract.

## Data Pipeline

ChunkFlag defines the states that drive a chunk through its data and processing pipeline. It acts as a gatekeeper and state signal at each stage.

> Flow:
> Chunk Request -> Chunk object created with **ChunkFlag.START_INIT** -> World Generator -> Chunk data populated, flag set to **ChunkFlag.INIT** and **ChunkFlag.NEWLY_GENERATED** -> World Ticker -> Ticker queries for chunks with **ChunkFlag.TICKING** and processes them -> World Serializer -> Chunk saved, flag set to **ChunkFlag.ON_DISK**

