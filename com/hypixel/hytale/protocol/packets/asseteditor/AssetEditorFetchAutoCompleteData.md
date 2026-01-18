---
description: Architectural reference for AssetEditorFetchAutoCompleteData
---

# AssetEditorFetchAutoCompleteData

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object (Packet)

## Definition
```java
// Signature
public class AssetEditorFetchAutoCompleteData implements Packet {
```

## Architecture & Concepts
The AssetEditorFetchAutoCompleteData class is a data structure that represents a network packet within the Hytale protocol. It is not a service or manager, but a message sent from the client to the server. Its specific purpose is to request auto-completion suggestions within the in-game asset editor. For example, when a user starts typing in a field, the client constructs and sends this packet to the server to fetch a list of possible values.

Architecturally, this class embodies a highly optimized serialization pattern common in high-performance networking. The binary layout is not a direct mapping of Java object memory but a custom format designed to minimize payload size and processing time.

The core design consists of two parts:
1.  **Fixed-Size Block:** A predictable header containing a bitmask for null fields (`nullBits`), a request identifier (`token`), and offsets pointing to the location of variable-length data.
2.  **Variable-Size Block:** A data region appended after the fixed block, containing the actual string content for `dataset` and `query`.

This structure allows the receiver to parse the fixed-size header and immediately know the full size of the packet and the location of all fields without scanning the entire buffer. The use of a `nullBits` byte as a bitmask is a space-saving technique to avoid sending data for optional fields.

## Lifecycle & Ownership
-   **Creation:**
    -   **Client-Side:** Instantiated by the UI logic of the asset editor in response to user input. The client application creates a new instance, populates the `token`, `dataset`, and `query` fields, and submits it to the network layer for transmission.
    -   **Server-Side:** Instantiated by the protocol's deserialization layer. The static `deserialize` method is invoked by a packet dispatcher when a byte buffer with the corresponding packet ID (331) is received from a client.

-   **Scope:** This object is ephemeral and has an extremely short lifespan. It exists only for the duration of a single client-server request/response cycle. It is created, serialized (on the client) or deserialized (on the server), processed by a handler, and then becomes eligible for garbage collection.

-   **Destruction:** Managed entirely by the Java Garbage Collector. Once the network layer has serialized the packet or the server-side handler has finished processing it, all references are dropped, and the object's memory is reclaimed.

## Internal State & Concurrency
-   **State:** The class is a mutable data container. Its public fields can be directly modified after instantiation. This design choice facilitates easy population by the application logic (client-side) and the deserialization process (server-side). It holds no external resources or handles.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and processed within a single thread, typically a Netty I/O worker thread. Sharing an instance of this packet across multiple threads without explicit, external synchronization is a critical anti-pattern and will lead to data corruption and unpredictable behavior.

## API Surface
The public contract is focused on serialization, deserialization, and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf buf) | void | O(N) | Encodes the object's state into the provided Netty ByteBuf according to the custom binary protocol. N is the total length of the string fields. |
| deserialize(ByteBuf buf, int offset) | AssetEditorFetchAutoCompleteData | O(N) | A static factory method that decodes a new instance from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the object. Useful for pre-allocating buffers. |
| validateStructure(ByteBuf buffer, int offset) | ValidationResult | O(1) | Performs a lightweight, read-only check of the packet structure in a buffer. **Crucially**, it validates offsets and lengths *before* attempting full deserialization to prevent buffer overflows or excessive memory allocation attacks. |
| getId() | int | O(1) | Returns the static network identifier for this packet type (331). |

## Integration Patterns

### Standard Usage
This packet is used to query the server for auto-complete data. The client creates the packet, sends it, and awaits a corresponding response packet.

**Client-Side Example:**
```java
// Executed in response to a UI event in the asset editor
int transactionToken = getNextToken();
String currentDataset = "hytale:materials";
String partialQuery = "sto";

AssetEditorFetchAutoCompleteData request = new AssetEditorFetchAutoCompleteData(
    transactionToken,
    currentDataset,
    partialQuery
);

// The network client will serialize and send the packet
networkClient.sendPacket(request);
```

**Server-Side Handling (Conceptual):**
```java
// Inside a packet handler for AssetEditorFetchAutoCompleteData
public void handleFetchRequest(AssetEditorFetchAutoCompleteData packet) {
    List<String> suggestions = autoCompleteService.getSuggestions(
        packet.dataset,
        packet.query
    );

    // Construct and send a response packet back to the client
    AssetEditorAutoCompleteResponse response = new AssetEditorAutoCompleteResponse(
        packet.token,
        suggestions
    );
    
    clientConnection.sendPacket(response);
}
```

### Anti-Patterns (Do NOT do this)
-   **Instance Re-use:** Do not cache and reuse packet instances. They are lightweight objects intended for single-use. Reusing an instance can lead to state corruption if not properly reset.
-   **Cross-Thread Access:** Never modify a packet's fields from one thread while another thread is serializing it. This will cause a race condition, potentially sending a partially updated and corrupt packet.
-   **Skipping Validation:** On the server, failing to call `validateStructure` on incoming data before `deserialize` exposes the system to denial-of-service vulnerabilities. A malicious client could send a packet with an invalid length field, causing the server to attempt a massive, fatal memory allocation.

## Data Pipeline
The flow of this data object is linear and unidirectional through the network stack.

> **Client Flow:**
> User Input in Asset Editor -> UI Event Handler -> **new AssetEditorFetchAutoCompleteData()** -> Network Manager -> `serialize()` -> Netty I/O Thread -> TCP Socket

> **Server Flow:**
> TCP Socket -> Netty I/O Thread -> Packet Framer/Decoder -> `validateStructure()` -> `deserialize()` -> **AssetEditorFetchAutoCompleteData instance** -> Packet Handler Logic -> Response Generation

