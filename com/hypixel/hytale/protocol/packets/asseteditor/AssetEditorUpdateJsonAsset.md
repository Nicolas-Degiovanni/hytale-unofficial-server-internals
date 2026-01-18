---
description: Architectural reference for AssetEditorUpdateJsonAsset
---

# AssetEditorUpdateJsonAsset

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object (Packet)

## Definition
```java
// Signature
public class AssetEditorUpdateJsonAsset implements Packet {
```

## Architecture & Concepts
The AssetEditorUpdateJsonAsset class is a specialized Data Transfer Object (DTO) designed for the Hytale network protocol. It represents a transactional update to a JSON-based game asset, forming a critical component of the live asset editing system.

This packet encapsulates a set of modifications, represented as an array of JsonUpdateCommand objects. This design follows the **Command Pattern**, where discrete editing operations (e.g., add field, modify value) are batched and sent from a client (the asset editor) to a server or preview client for application.

Its primary role is to serve as a structured, serializable container for data in transit. The class itself contains the complete logic for its own binary serialization and deserialization, adhering to a custom, performance-oriented wire format. This format utilizes a fixed-size header containing offsets to a variable-sized data block, an efficient strategy for parsing complex network messages.

## Lifecycle & Ownership
As a transient DTO, an AssetEditorUpdateJsonAsset instance has a very short and well-defined lifecycle tied directly to a single network operation.

-   **Creation:**
    -   **Sending:** Instantiated directly by a high-level system (e.g., an Asset Editor service) that needs to transmit changes. The constructor is populated with the relevant asset path and a list of commands.
    -   **Receiving:** Instantiated by the protocol layer via the static *deserialize* method when an incoming byte buffer with Packet ID 323 is detected.
-   **Scope:** The object's lifetime is confined to the scope of a single network event processing block. It is created, used to transfer data to or from a byte buffer, processed by a handler, and then immediately becomes eligible for garbage collection.
-   **Destruction:** There is no explicit destruction logic. The Java Garbage Collector reclaims the memory once all references are dropped, typically upon exiting the network handler method. **WARNING:** Holding references to packet objects after processing is a potential memory leak.

## Internal State & Concurrency
-   **State:** The class is **highly mutable**. All of its fields are public and can be modified after construction. This design facilitates the direct memory mapping performed during deserialization, where fields are populated one by one from a ByteBuf. The state represents a snapshot of a requested asset modification at a specific point in time.
-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads without explicit, external synchronization. It is designed to be created, serialized, deserialized, and processed within a single network thread (e.g., a Netty I/O worker thread) to guarantee memory visibility and prevent data corruption.

## API Surface
The public contract is dominated by the static methods required by the protocol framework for handling the binary wire format.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | AssetEditorUpdateJsonAsset | O(N) | **(Receiver)** Constructs an object by reading from a ByteBuf. N is the size of the variable data. |
| serialize(buf) | void | O(N) | **(Sender)** Writes the object's state into a ByteBuf. N is the size of the variable data. |
| computeSize() | int | O(N) | Calculates the total byte size the serialized object will occupy. N is the number of commands. |
| validateStructure(buf, offset) | ValidationResult | O(N) | Performs sanity checks on a raw ByteBuf to ensure it represents a valid packet before deserialization. |
| clone() | AssetEditorUpdateJsonAsset | O(N) | Creates a deep copy of the packet and its contained commands. |

## Integration Patterns

### Standard Usage
The class is intended to be used by the network protocol layer. A handler receives the deserialized object, inspects its contents, and dispatches the commands to the relevant asset management system.

```java
// Example of a packet handler processing the update
void handlePacket(AssetEditorUpdateJsonAsset packet) {
    // Retrieve the target asset using information from the packet
    JsonAsset targetAsset = AssetRepository.get(packet.assetType, packet.path);

    if (targetAsset != null && packet.commands != null) {
        // Apply the batch of changes to the asset
        targetAsset.applyUpdateCommands(packet.commands);
        System.out.println("Successfully updated asset.");
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term Storage:** Do not store instances of this packet in caches or as part of a persistent game state. It is a transient message, not a data model.
-   **Cross-Thread Modification:** Do not deserialize a packet on a network thread and then pass the mutable object to a worker thread for modification without proper locking or creating a defensive copy. This will lead to race conditions.
-   **Ignoring Validation:** Bypassing *validateStructure* on untrusted input exposes the system to potential buffer overflows or denial-of-service attacks from malformed packets.

## Data Pipeline
The AssetEditorUpdateJsonAsset acts as a data record flowing through the asset editing network pipeline.

> **Flow (Client to Server):**
> User Action in Editor -> Generates JsonUpdateCommand[] -> **new AssetEditorUpdateJsonAsset()** -> serialize() -> Netty Channel -> Network -> Server Packet Decoder -> **deserialize()** -> Packet Handler -> AssetManager -> Live Game Asset Update

