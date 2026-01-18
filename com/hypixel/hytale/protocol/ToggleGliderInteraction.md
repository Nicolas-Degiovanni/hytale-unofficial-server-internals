---
description: Architectural reference for ToggleGliderInteraction
---

# ToggleGliderInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class ToggleGliderInteraction extends SimpleInteraction {
```

## Architecture & Concepts
The ToggleGliderInteraction class is a Data Transfer Object (DTO) that represents a specific, serializable game action within the Hytale network protocol. It is not a service or a manager; its sole responsibility is to encapsulate the state required to define the behavior of a player toggling their glider.

This class is a fundamental component of the **Protocol Layer**. It serves as a concrete message contract between the client and server. Its structure is highly optimized for network efficiency, employing a custom serialization format that minimizes payload size.

The serialization strategy is a key architectural feature:
1.  **Null Bit Field:** The first byte of the payload is a bitmask (`nullBits`) indicating which of the object's nullable fields are present in the data stream. This avoids wasting bytes for absent optional data.
2.  **Fixed-Size Block:** A block of a known, fixed size (`FIXED_BLOCK_SIZE`) immediately follows the null bit field. It contains primitive types and fixed-size data like floats, booleans, and integers.
3.  **Variable-Data Offsets:** Within the fixed-size block, there are integer offsets that point to the location of variable-sized data (such as maps or arrays) within the payload.
4.  **Variable-Data Block:** All variable-sized data is appended after the fixed-size block. This layout prevents parsing ambiguity and allows for efficient, direct memory access during deserialization.

This design ensures that the protocol is both compact and forward-compatible, as new optional fields can be added by assigning new bits in the null field without breaking older clients.

## Lifecycle & Ownership
- **Creation:** An instance of ToggleGliderInteraction is created under two distinct circumstances:
    1.  **Serialization (Outgoing):** Instantiated directly via its constructor by game logic on the sending side (e.g., the server) when this interaction needs to be communicated.
    2.  **Deserialization (Incoming):** Instantiated by the protocol layer via the static `deserialize` factory method when a corresponding network packet is received.
- **Scope:** The object's lifetime is exceptionally brief and transactional. It exists only long enough to be serialized into a network buffer or to be processed after being deserialized from one. It does not persist between game ticks or network events.
- **Destruction:** The object becomes eligible for garbage collection immediately after its data has been written to a `ByteBuf` or read and applied to the game state. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The internal state is entirely mutable and represents a snapshot of the interaction's parameters at a single point in time. The object acts as a simple data container with no internal logic beyond serialization and validation.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and processed within the confines of a single thread, typically a Netty I/O thread or the main game logic thread.
    - **WARNING:** Sharing an instance across multiple threads without external synchronization will lead to race conditions and undefined behavior. Such usage is a critical anti-pattern.

## API Surface
The public API is dominated by static methods for creating instances from raw network data and instance methods for writing to a network buffer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | ToggleGliderInteraction | O(N) | **[Factory]** Constructs an object by parsing a network buffer. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | int | O(N) | Writes the object's state to a network buffer using the custom binary format. Returns bytes written. |
| validateStructure(ByteBuf, int) | ValidationResult | O(N) | Performs a structural pre-validation on a buffer without full deserialization. Crucial for security and stability. |
| computeSize() | int | O(N) | Calculates the exact byte size the object will occupy when serialized. |
| computeBytesConsumed(ByteBuf, int) | int | O(N) | Calculates the size of an already-serialized object within a buffer. Used for advancing buffer read pointers. |

## Integration Patterns

### Standard Usage
This object is almost exclusively handled by the network protocol machinery. A developer would typically interact with it after it has been deserialized and dispatched as a game event.

```java
// Example of how the network layer would process an incoming buffer
// This code would exist deep within the protocol handling logic.

ByteBuf incomingPacket = ...;
int offset = ...; // Start of the interaction data

// 1. Validate before parsing to prevent errors and exploits
ValidationResult result = ToggleGliderInteraction.validateStructure(incomingPacket, offset);
if (!result.isValid()) {
    throw new ProtocolException("Invalid ToggleGliderInteraction: " + result.error());
}

// 2. Deserialize into a usable object
ToggleGliderInteraction interaction = ToggleGliderInteraction.deserialize(incomingPacket, offset);

// 3. Dispatch to the game engine for processing
GameEventHandler.handle(interaction);
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not deserialize an object, modify its fields, and then re-serialize it. This can lead to inconsistent state. Always create a new, clean instance for outgoing messages.
- **Manual Buffer Manipulation:** Never attempt to read or write fields directly from the `ByteBuf`. The serialization format is complex, involving offsets and bitmasks. Always use the `serialize` and `deserialize` methods to ensure correctness.
- **Ignoring Validation:** Bypassing `validateStructure` before deserialization is dangerous. A malformed packet could trigger exceptions, buffer over-reads, or denial-of-service vulnerabilities.

## Data Pipeline
The ToggleGliderInteraction serves as the data payload that flows from the game logic on one end of the connection to the game logic on the other.

> **Flow (Server to Client):**
> Game Logic Event -> `new ToggleGliderInteraction(...)` -> **`serialize()`** -> Netty `ByteBuf` -> TCP/IP Stack -> Client Network Layer -> **`deserialize()`** -> Game State Update -> Player's glider activates

---

