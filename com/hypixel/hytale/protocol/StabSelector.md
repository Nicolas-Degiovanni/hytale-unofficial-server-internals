---
description: Architectural reference for StabSelector
---

# StabSelector

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Transfer Object (DTO)

## Definition
```java
// Signature
public class StabSelector extends Selector {
```

## Architecture & Concepts

The StabSelector is a specialized, fixed-size data structure that represents a specific type of targeting volume within the game world. As a subclass of Selector, its primary function is to define the parameters for selecting entities, but its name and fields strongly imply its use in the combat system to define the geometry of a melee "stab" or thrust attack.

Architecturally, this class serves as a high-performance Data Transfer Object (DTO) within the network protocol layer. Its design prioritizes raw speed and a predictable memory layout over encapsulation. The structure is rigidly defined to be exactly 37 bytes, allowing for extremely fast serialization and deserialization directly to and from a network buffer.

Key architectural characteristics include:
*   **Fixed-Size Payload:** The class always consumes and produces exactly 37 bytes, eliminating the need for size prefixes or complex parsing logic. This is critical for performance in the server's network processing pipeline.
*   **Direct Memory Mapping:** The `serialize` and `deserialize` methods perform direct, low-level reads and writes of primitive types (floats and a byte) from a Netty ByteBuf. This avoids intermediate object creation and reflection, minimizing garbage collection pressure.
*   **Little-Endian Byte Order:** All multi-byte numeric fields are processed using little-endian byte order, a crucial detail for ensuring cross-platform compatibility between the client and server.

This class is a fundamental building block for network-synchronized combat, acting as the precise contract for how a client communicates the intent and shape of an attack to the server.

## Lifecycle & Ownership

-   **Creation:** StabSelector instances are created in two primary scenarios:
    1.  **Deserialization:** The static `deserialize` factory method is called by the network protocol decoder when a corresponding packet is read from the wire. This creates a new StabSelector populated with data from a remote peer.
    2.  **Game Logic:** The game's combat system instantiates a StabSelector (typically via the all-arguments constructor) to define a new attack that is about to be performed and sent over the network.

-   **Scope:** Instances are extremely short-lived and transaction-scoped. A StabSelector typically exists only for the duration of processing a single network packet or a single game tick. It is a value object, not a persistent entity.

-   **Destruction:** The object becomes eligible for garbage collection as soon as the network or game logic that created it completes its operation. There is no manual memory management or destruction process.

## Internal State & Concurrency

-   **State:** The class is fully mutable, with all fields exposed as public members. This design choice prioritizes direct, high-speed access for the network and game logic threads, sacrificing data integrity guarantees that would be provided by encapsulation. It contains no internal caches or derived state.

-   **Thread Safety:** **This class is not thread-safe.** Its public mutable fields make it inherently unsafe for concurrent access. It is designed to be created, manipulated, and read exclusively within a single, well-defined thread context, such as a Netty I/O worker thread or the main server game loop thread.

    **WARNING:** Sharing a StabSelector instance across threads without external locking will lead to race conditions, data corruption, and unpredictable behavior. Such a pattern is strictly forbidden.

## API Surface

The public API is dominated by static utility methods for network I/O and instance methods for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | StabSelector | O(1) | **[Factory]** Constructs a new StabSelector by reading 37 bytes from the given buffer at the specified offset. |
| serialize(ByteBuf) | int | O(1) | Writes the object's 37-byte state into the provided buffer. Returns the number of bytes written. |
| computeSize() | int | O(1) | Returns the constant size of the structure, which is always 37. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | Performs a pre-check to ensure the buffer contains enough readable bytes for a successful deserialization. |
| clone() | StabSelector | O(1) | Creates a shallow copy of the object. Useful for creating a mutable copy from a template. |

## Integration Patterns

### Standard Usage

The primary interaction pattern involves the static `deserialize` method for inbound data or creating an instance and passing it to a serializer for outbound data.

```java
// Example: Deserializing from a network buffer
// Assume 'packetBuffer' is a ByteBuf received from the network.
ValidationResult result = StabSelector.validateStructure(packetBuffer, 0);

if (result.isOk()) {
    StabSelector attackVolume = StabSelector.deserialize(packetBuffer, 0);
    
    // The combat system can now use the attackVolume
    // to determine which entities were hit.
    gameWorld.processMeleeAttack(attacker, attackVolume);
}
```

### Anti-Patterns (Do NOT do this)

-   **Instance Re-use:** Do not reuse a StabSelector instance across multiple network packets or game ticks. Its state is specific to a single event. Caching or pooling these objects is unnecessary and error-prone.
-   **Concurrent Modification:** Never write to a StabSelector from one thread while another thread is reading it or writing it to a network buffer. All operations on a given instance must be confined to a single thread.
-   **Ignoring Buffer Position:** The `deserialize` and `validateStructure` methods operate on an absolute offset. It is the caller's responsibility to manage the read and write indices of the ByteBuf. Incorrectly managing these offsets can lead to data corruption.

## Data Pipeline

The StabSelector is a data payload that flows through the network and combat systems. Its path is simple and direct.

**Inbound Flow (Client Attack -> Server)**
> Flow:
> Network ByteBuf -> Protocol Decoder -> **StabSelector.deserialize** -> Combat System -> World State Update

**Outbound Flow (Server Action -> Client)**
> Flow:
> Game Logic -> **new StabSelector(...)** -> Protocol Encoder -> **StabSelector.serialize** -> Network ByteBuf

