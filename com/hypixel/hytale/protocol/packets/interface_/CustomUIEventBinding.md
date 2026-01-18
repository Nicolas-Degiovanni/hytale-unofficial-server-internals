---
description: Architectural reference for CustomUIEventBinding
---

# CustomUIEventBinding

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class CustomUIEventBinding {
```

## Architecture & Concepts

The CustomUIEventBinding class is a specialized Data Transfer Object (DTO) designed for high-performance network communication. It represents a discrete user interaction event originating from a custom UI, such as a button click or a form submission. Its primary role is to serve as a structured, serializable container for event data transmitted between the client and server.

This class is not a service or a manager; it is a fundamental building block of the Hytale network protocol layer. The design prioritizes serialization speed and compact binary representation over runtime flexibility.

The binary layout is a key architectural feature, optimized to minimize parsing overhead. It employs a hybrid fixed-size and variable-size block structure:

1.  **Nullable Bit Field (1 byte):** A bitmask indicating which of the nullable fields, such as *selector* and *data*, are present in the payload. This avoids wasting space for null values.
2.  **Fixed-Size Block (10 bytes):** Contains the bit field, the event *type*, the *locksInterface* boolean, and two 4-byte integer offsets for the variable-sized fields.
3.  **Variable-Size Block:** Contains the actual string content for *selector* and *data*. The offsets in the fixed-size block point to the start of each string within this block, allowing for direct memory access without sequential parsing.

This structure is highly efficient for network deserializers, which can read the fixed-size header to immediately understand the full size and layout of the packet payload.

## Lifecycle & Ownership

-   **Creation:** An instance is created under two primary circumstances:
    1.  **Deserialization:** The static `deserialize` method is invoked by the network protocol layer when an incoming packet of the corresponding type is received. The network layer owns the initial `ByteBuf` and the method returns a new `CustomUIEventBinding` instance.
    2.  **Manual Instantiation:** Game logic (e.g., the UI system) creates a new instance via its constructor to prepare an outgoing event for transmission.

-   **Scope:** The object is ephemeral and has a very short lifecycle. It exists only for the duration of processing a single network packet or for the assembly of an outgoing packet. It is not intended to be stored or referenced long-term.

-   **Destruction:** The object is managed by the Java Garbage Collector. Once all references are dropped (typically after a network handler has finished processing it or after it has been written to a network buffer), it becomes eligible for collection. No manual cleanup is required.

## Internal State & Concurrency

-   **State:** The class is a mutable container for event data. Its fields are public and can be modified after construction. This design facilitates easy population of the object before serialization.

-   **Thread Safety:** **This class is not thread-safe.** It is a simple data structure with no internal locking or synchronization. It is designed to be created, populated, and read within the confines of a single thread, such as a Netty event loop thread or the main game thread.

    **WARNING:** Concurrent access from multiple threads will lead to race conditions and unpredictable behavior, especially if one thread is serializing the object while another is modifying its fields. All interaction with an instance must be externally synchronized or confined to a single thread.

## API Surface

The public API is focused entirely on serialization, deserialization, and validation of the network representation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | CustomUIEventBinding | O(N) | **[Static]** Constructs a new object by reading from a ByteBuf. Throws ProtocolException on malformed data. N is the size of the variable string data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the defined binary layout. N is the size of the variable string data. |
| validateStructure(buf, offset) | ValidationResult | O(N) | **[Static]** Performs a safe, read-only check of the binary data in a buffer to ensure it represents a valid object. Does not throw. Crucial for security and stability. |
| computeSize() | int | O(N) | Calculates the total number of bytes the object will consume when serialized. Useful for pre-allocating buffers. N is the size of the variable string data. |
| computeBytesConsumed(buf, offset) | int | O(N) | **[Static]** Calculates the size of an already-serialized object within a buffer by reading its headers and variable field lengths. |

## Integration Patterns

### Standard Usage

The class is used by the network layer and game systems to send and receive UI events. The typical flow involves creating an object, populating it, and passing it to an encoder, or receiving an object from a decoder and processing its contents.

```java
// Example: Sending a UI click event from the client
CustomUIEventBinding clickEvent = new CustomUIEventBinding(
    CustomUIEventBindingType.Activating,
    "#profile.friendsButton",
    null, // No extra data for this click
    false
);

// The event object is then wrapped in a packet and sent
// to the network layer for serialization and transmission.
networkManager.sendPacket(new CustomUIPacket(clickEvent));
```

### Anti-Patterns (Do NOT do this)

-   **Instance Re-use:** Do not hold onto and re-use a `CustomUIEventBinding` instance for multiple distinct events. This is error-prone. Always create a new instance for each new event to ensure state integrity.
-   **Concurrent Modification:** Do not modify an instance from one thread while it is being serialized or read by another. This will corrupt the network stream or cause deserialization failures.
-   **Skipping Validation:** Never call `deserialize` on a buffer received from an untrusted source (i.e., any remote client) without first calling `validateStructure`. Bypassing validation can lead to `ProtocolException` crashes or, in worse cases, excessive memory allocation vulnerabilities.

## Data Pipeline

The `CustomUIEventBinding` class is a critical link in the data flow between raw network bytes and actionable game logic.

**Inbound Flow (Receiving an Event):**
> Raw Bytes (Netty ByteBuf) -> Protocol Decoder -> **CustomUIEventBinding.deserialize()** -> CustomUIEventBinding Object -> UI Packet Handler -> Game Event Bus

**Outbound Flow (Sending an Event):**
> Game Logic (UI System) -> CustomUIEventBinding Object -> **CustomUIEventBinding.serialize()** -> Protocol Encoder -> Raw Bytes (Netty ByteBuf) -> Network Transmission

