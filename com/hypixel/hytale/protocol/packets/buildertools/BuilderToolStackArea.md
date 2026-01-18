---
description: Architectural reference for BuilderToolStackArea
---

# BuilderToolStackArea

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class BuilderToolStackArea implements Packet {
```

## Architecture & Concepts
The BuilderToolStackArea class is a network **Packet**, a specialized Data Transfer Object (DTO) designed for client-server communication. It does not contain any business logic. Instead, it serves as a structured, serializable message that represents a specific in-game action: performing a "stack" operation with a builder tool.

This packet is a fundamental component of the Hytale Protocol Layer. It acts as a data contract, defining the precise information required by the server to replicate a player's building command. Its structure is fixed-size for high-performance serialization and deserialization, which is critical for real-time gameplay.

The key concepts encapsulated by this packet are:
*   **Selection Volume:** A bounding box defined by *selectionMin* and *selectionMax* that specifies the source area of blocks to be copied.
*   **Stack Vector:** A direction defined by *xNormal*, *yNormal*, and *zNormal* along which the source volume will be repeatedly pasted.
*   **Stack Count:** The *numStacks* field indicates how many times the operation should be repeated.

## Lifecycle & Ownership
-   **Creation:** An instance of BuilderToolStackArea is created on the client when a player executes the corresponding build action. On the server (or a receiving client), it is instantiated exclusively by the protocol layer's packet decoder via the static *deserialize* method when a network buffer with Packet ID 404 is read.
-   **Scope:** This object is extremely short-lived and transaction-scoped. It exists only long enough to be serialized into a network buffer or, upon receipt, to be read by a packet handler. It is not managed, cached, or persisted by any long-term system.
-   **Destruction:** The object becomes eligible for garbage collection immediately after its data has been processed by a packet handler or written to the network channel. There is no explicit cleanup mechanism.

## Internal State & Concurrency
-   **State:** The object's state is fully mutable, with all fields being public. It is designed to be populated once and then treated as immutable for the remainder of its brief lifecycle. It contains no internal caches or derived data.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be created, serialized, and processed on a single thread, typically a Netty I/O thread or a main game-loop thread. Sharing an instance across multiple threads without explicit, external synchronization will result in undefined behavior and data corruption. The protocol framework guarantees single-threaded handling.

## API Surface
The public contract is dominated by serialization and object-utility methods, reflecting its role as a DTO.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf) | void | O(1) | Encodes the packet's state into the provided network buffer. This is a primary entry point for network transmission. |
| deserialize(ByteBuf, int) | static BuilderToolStackArea | O(1) | Decodes a network buffer into a new instance. This is the primary entry point for network reception. |
| computeSize() | int | O(1) | Returns the constant size (41 bytes) of the packet on the wire. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-flight check to ensure a buffer is large enough to contain this packet, preventing buffer over-reads. |
| getId() | int | O(1) | Returns the unique network identifier (404) for this packet type. |

## Integration Patterns

### Standard Usage
This packet should only be handled within a dedicated packet processing system. The handler extracts the data and forwards it to the appropriate game system, such as the world or entity manager, to execute the command.

```java
// Hypothetical server-side packet handler
public void onBuilderToolStackArea(BuilderToolStackArea packet) {
    Player sender = getContext().getPlayer();
    World world = sender.getWorld();

    // Validate the operation before executing
    if (!world.hasPermission(sender, packet.selectionMin)) {
        return; // Or send an error response
    }

    // Delegate the actual game logic to the world simulation
    world.performStackOperation(
        packet.selectionMin,
        packet.selectionMax,
        new Vector3i(packet.xNormal, packet.yNormal, packet.zNormal),
        packet.numStacks
    );
}
```

### Anti-Patterns (Do NOT do this)
-   **State Reuse:** Do not hold a reference to a BuilderToolStackArea packet after it has been processed. It represents a single, point-in-time event and should be discarded. Modifying and re-processing the same instance is an error-prone practice.
-   **Manual Deserialization:** Never use `new BuilderToolStackArea()` and manually read from a ByteBuf. The static `deserialize` method is the only correct way to construct an object from a network stream, as it properly handles the internal null-tracking bitfield.
-   **Long-Term Storage:** This object is not designed for persistence. Storing it in a database or holding it in a long-lived cache is an incorrect use of a network DTO.

## Data Pipeline
The flow of this data object is linear and unidirectional through the network stack.

> **Client Flow:**
> Player Input -> Builder Tool Logic -> **new BuilderToolStackArea(...)** -> Protocol Encoder -> `serialize()` -> Network Channel

> **Server Flow:**
> Network Channel -> Protocol Decoder -> `deserialize()` -> **BuilderToolStackArea instance** -> Packet Handler -> World Simulation Logic

