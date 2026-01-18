---
description: Architectural reference for WorldLoadProgress
---

# WorldLoadProgress

**Package:** com.hypixel.hytale.protocol.packets.setup
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class WorldLoadProgress implements Packet {
```

## Architecture & Concepts
The WorldLoadProgress class is a pure Data Transfer Object representing a single, specific network message. It is part of the *setup* protocol group, used during the client-server connection handshake and initial world loading sequence.

Its sole responsibility is to encapsulate the state of a world loading operation, typically sent from a server to a client. This allows the client to render a meaningful progress bar or status message on the loading screen. As an implementation of the Packet interface, it defines a strict binary serialization and deserialization contract, ensuring that both client and server can interpret the network data identically.

This class contains no business logic. It is a passive data structure manipulated by the network layer and the game state systems that produce or consume its information.

## Lifecycle & Ownership
- **Creation:**
    - **Sending Peer (Server):** Instantiated directly via its constructor (`new WorldLoadProgress(...)`) when the server needs to report a progress update. The object is populated with current loading data.
    - **Receiving Peer (Client):** Never instantiated with its constructor. The static factory method `deserialize` is invoked by the protocol's packet dispatcher when a raw ByteBuf with Packet ID 21 is received from the network.
- **Scope:** Extremely short-lived and transient. An instance exists only for the duration of one of two operations:
    1. From creation to serialization into a network buffer for sending.
    2. From deserialization out of a network buffer to the point where its data is consumed by a client-side system (e.g., the UI thread).
- **Destruction:** The object becomes eligible for garbage collection immediately after its data has been serialized or consumed. There are no manual cleanup or `close` methods.

## Internal State & Concurrency
- **State:** Highly mutable. All fields are public and can be modified after construction. The class is a simple container for three data points: `status`, `percentComplete`, and `percentCompleteSubitem`. It holds no other state and performs no caching.
- **Thread Safety:** **This class is not thread-safe.** It is designed for use within a single-threaded context, such as a Netty event loop or a main game thread. Accessing or modifying an instance from multiple threads simultaneously will result in unpredictable behavior and data corruption. All synchronization must be handled by the calling systems.

## API Surface
The public API is dominated by methods related to the Packet contract for network serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | WorldLoadProgress | O(N) | **Static Factory.** Reads binary data from a ByteBuf and constructs a new WorldLoadProgress instance. N is the length of the status string. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Encodes the object's state into the provided ByteBuf according to the protocol specification. N is the length of the status string. |
| computeSize() | int | O(N) | Calculates the exact number of bytes this packet will occupy on the network after serialization. Used for pre-allocating buffers. |
| validateStructure(buf, offset) | ValidationResult | O(N) | **Static.** Performs a read-only check on a buffer to determine if it contains a valid, non-corrupt WorldLoadProgress packet without fully deserializing it. |

## Integration Patterns

### Standard Usage
This packet is processed by a network event handler. The handler receives the deserialized object and typically dispatches an event or calls a subsystem to update the user interface.

```java
// In a client-side network handler
public void handle(WorldLoadProgress packet) {
    // Extract data from the transient packet object
    String status = packet.status;
    int percent = packet.percentComplete;

    // Pass the data to the UI or game state manager
    // The 'packet' object can now be garbage collected
    GameUI.updateLoadingScreen(status, percent);
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Reuse:** Do not hold references to WorldLoadProgress objects long-term or attempt to reuse them for subsequent messages. They are cheap to create and should be treated as immutable after deserialization.
- **Cross-Thread Modification:** Do not deserialize a packet on a network thread and modify its fields on a game thread. Pass the raw data (status, percent) between threads, not the mutable packet object itself.
- **Manual Deserialization:** Do not attempt to read the fields from a ByteBuf manually. Always use the provided static `deserialize` method to ensure correctness and handle protocol details like null-bit fields and VarInts.

## Data Pipeline
The class acts as a data container in a simple, linear pipeline.

> **Server Flow:**
> World Generation System -> **new WorldLoadProgress()** -> Packet Serializer -> Netty Channel -> Client

> **Client Flow:**
> Server -> Netty Channel -> Packet Dispatcher -> **WorldLoadProgress.deserialize()** -> Client Network Handler -> UI System

