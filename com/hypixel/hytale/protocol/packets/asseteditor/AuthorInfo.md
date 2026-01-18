---
description: Architectural reference for AuthorInfo
---

# AuthorInfo

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class AuthorInfo {
```

## Architecture & Concepts
The AuthorInfo class is a specialized Data Transfer Object (DTO) designed for the high-performance serialization and deserialization of asset author metadata within Hytale's network protocol. It is not a service or a managed component; it is a pure data structure that represents a well-defined binary layout.

The core design prioritizes network efficiency and explicit control over byte-level representation, eschewing generic serialization frameworks like JSON or Protobuf for a custom format. This format is characterized by:

-   **Fixed-Size Header:** The structure always begins with a predictable 13-byte block.
-   **Nullability Bitmask:** The first byte of the header is a bitmask, where each bit corresponds to a nullable field (name, email, url). This allows the system to determine which fields are present without parsing the entire object, enabling efficient skipping of data.
-   **Offset-Based Pointers:** The header contains 4-byte little-endian integer offsets that point to the start of each variable-length string in the subsequent data block. This design permits non-sequential data access and resilience to changes in field order.
-   **Variable-Length Data Block:** All string data is appended after the fixed-size header. This separation of metadata (header) and payload (data) is a common pattern in high-throughput network protocols.

This class acts as a direct contract between the game client and server for how author information is encoded on the wire.

## Lifecycle & Ownership
-   **Creation:** An AuthorInfo instance is created under two primary circumstances:
    1.  **Inbound:** The static `deserialize` factory method constructs an instance from a raw Netty ByteBuf, typically within a network packet decoder.
    2.  **Outbound:** Game logic, such as an asset editor preparing to save or transmit data, instantiates it directly using its constructors (`new AuthorInfo(...)`).
-   **Scope:** The object is **transient and short-lived**. Its lifecycle is typically confined to the scope of a single network packet processing routine. It is not managed by a dependency injection container or a service registry.
-   **Destruction:** The object is managed by the Java Garbage Collector. It holds no native resources and requires no explicit cleanup. Once all references to an instance are gone, it is eligible for collection.

## Internal State & Concurrency
-   **State:** The internal state consists of three public, nullable String fields: name, email, and url. The class is fundamentally **mutable**, and its fields can be modified directly at any time after instantiation. It performs no internal caching.
-   **Thread Safety:** This class is **not thread-safe**. Direct public access to its fields makes it vulnerable to race conditions if shared between threads without external synchronization.

    **WARNING:** All operations on an AuthorInfo instance must be confined to a single thread, typically the Netty I/O thread that is processing the network packet. Do not pass instances to worker threads without ensuring safe publication and synchronized access.

## API Surface
The public API is dominated by static utility methods for interacting with the binary protocol format.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static AuthorInfo | O(N) | Constructs an AuthorInfo object by reading from a buffer at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the object's state into the provided buffer according to the defined binary format. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a read-only check of the buffer to ensure it contains a structurally valid AuthorInfo object. Does not fully deserialize. |
| computeBytesConsumed(ByteBuf, int) | static int | O(N) | Calculates the total size in bytes of a serialized AuthorInfo object within a buffer without full deserialization. |
| computeSize() | int | O(N) | Calculates the number of bytes this object will occupy when serialized. |

*N = total length of string data*

## Integration Patterns

### Standard Usage
AuthorInfo is used either to encode data for sending or to decode received data. It is almost always handled within the context of a larger network packet.

```java
// Example: Deserializing from a received buffer
// This code would typically exist inside a packet handler.

ByteBuf incomingBuffer = ...;
ValidationResult result = AuthorInfo.validateStructure(incomingBuffer, 0);

if (result.isOk()) {
    AuthorInfo author = AuthorInfo.deserialize(incomingBuffer, 0);
    System.out.println("Asset created by: " + author.name);
} else {
    // Handle or log the validation error
    System.err.println("Invalid AuthorInfo data: " + result.getErrorMessage());
}
```

### Anti-Patterns (Do NOT do this)
-   **State Reuse:** Do not reuse an AuthorInfo instance across multiple operations without explicitly clearing its fields. Its mutable nature makes it easy to leak old data into new packets.
    ```java
    // BAD: Leaks "old.author.com" into the second packet
    AuthorInfo author = new AuthorInfo("Player1", "p1@hytale.com", "old.author.com");
    packet1.setAuthor(author);
    
    author.name = "Player2";
    author.email = "p2@hytale.com";
    // The 'url' field is not cleared and remains from the previous state
    packet2.setAuthor(author); 
    ```
-   **Ignoring Validation:** Directly calling `deserialize` on untrusted network data without a prior call to `validateStructure` can result in uncaught ProtocolExceptions, potentially terminating the connection handler. Always validate first.
-   **Concurrent Modification:** Sharing an instance across threads for read/write operations without external locking will lead to data corruption and unpredictable behavior.

## Data Pipeline
AuthorInfo serves as a serialization endpoint in the data flow. It translates between the in-memory object representation and the on-the-wire binary format.

**Outbound Flow (Serialization):**
> Game Logic (e.g., Asset Editor) -> `new AuthorInfo(...)` -> **AuthorInfo.serialize(buffer)** -> Network Packet Encoder -> TCP Socket

**Inbound Flow (Deserialization):**
> TCP Socket -> Netty ByteBuf -> Network Packet Decoder -> **AuthorInfo.deserialize(buffer)** -> Packet Handler -> Game Logic

