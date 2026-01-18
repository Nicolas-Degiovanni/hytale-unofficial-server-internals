---
description: Architectural reference for FluidChange
---

# FluidChange

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class FluidChange {
```

## Architecture & Concepts
The FluidChange class is a low-level, high-performance data structure that represents a single, atomic update to a fluid block within the game world. It is not a service or manager, but rather a fundamental component of the network protocol's data serialization layer.

Its primary role is to act as a "packet struct", a plain data container designed for efficient serialization to and deserialization from a network byte stream. The class operates directly on Netty's ByteBuf, indicating its position deep within the networking stack, where performance is critical.

The design emphasizes speed and minimal overhead:
*   **Fixed-Size Layout:** The object has a constant, predictable size of 17 bytes, allowing for extremely fast serialization and memory allocation without complex calculations.
*   **Public Fields:** Direct field access is used instead of getter and setter methods to eliminate method call overhead, a common optimization in performance-sensitive game engine code.
*   **Struct-like Behavior:** It behaves like a C-style struct, holding a self-contained, value-based representation of a world event.

This class is a foundational element for synchronizing the state of in-game fluids between the server and clients.

## Lifecycle & Ownership
- **Creation:** FluidChange objects are instantiated under two primary circumstances:
    1.  **Outbound (Serialization):** The server-side game logic creates an instance to represent a change in the world (e.g., fluid flowing) before serializing it into a larger packet.
    2.  **Inbound (Deserialization):** The client-side network layer creates an instance via the static *deserialize* method after receiving raw byte data from the server.
- **Scope:** The lifecycle of a FluidChange instance is extremely short and transient. It is designed to exist only for the duration of a single network operation or game tick processing cycle.
- **Destruction:** Instances are managed by the Java Garbage Collector. Due to their short scope, they are typically reclaimed quickly. There are no manual resource management or cleanup requirements.

## Internal State & Concurrency
- **State:** The object is fully **mutable**. Its public fields are intended for direct modification after creation. It contains no internal logic, only the data representing the fluid state change.
- **Thread Safety:** This class is **not thread-safe**. It is designed for use within a single, well-defined thread, such as a Netty I/O worker thread or the main game loop thread.

    **WARNING:** Sharing a FluidChange instance across multiple threads without explicit, external synchronization will lead to race conditions and unpredictable data corruption. Do not pass instances between threads; instead, pass the primitive data it contains.

## API Surface
The public API is minimal, focusing exclusively on serialization and data validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static FluidChange | O(1) | Constructs a new FluidChange object by reading 17 bytes from the provided buffer at a specific offset. |
| serialize(ByteBuf) | void | O(1) | Writes the object's 17 bytes of state into the provided buffer using little-endian byte order. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a bounds check to ensure the buffer contains at least 17 readable bytes from the given offset. |
| computeSize() | int | O(1) | Returns the fixed size of the serialized object, which is always 17. |

## Integration Patterns

### Standard Usage
The class is used as part of a larger packet processing pipeline. The server creates and serializes it, while the client deserializes and consumes it.

```java
// SERVER: Creating and serializing a fluid change
FluidChange change = new FluidChange(10, 64, -20, WATER_FLUID_ID, (byte) 7);
ByteBuf networkBuffer = ...; // Obtain buffer from Netty channel
change.serialize(networkBuffer);

// CLIENT: Deserializing and processing a fluid change
ByteBuf receivedBuffer = ...; // Buffer from incoming packet
if (FluidChange.validateStructure(receivedBuffer, offset).isOk()) {
    FluidChange receivedChange = FluidChange.deserialize(receivedBuffer, offset);
    // Apply the change to the client's world representation
    world.setFluid(receivedChange.x, receivedChange.y, receivedChange.z, receivedChange.fluidId, receivedChange.fluidLevel);
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not retain references to FluidChange objects in caches or as part of the persistent world state. They are transient DTOs. Copy their primitive data into your primary world data structures immediately after deserialization.
- **Cross-Thread Sharing:** Never share an instance of FluidChange between threads. Its mutable, non-synchronized nature makes it inherently unsafe for concurrent access.
- **Manual Deserialization:** Do not use `new FluidChange()` and then manually read fields from a ByteBuf. This bypasses the official `deserialize` method, which correctly handles byte order (little-endian) and field offsets.

## Data Pipeline
FluidChange is a data payload that flows through the network stack. It does not initiate operations but is instead acted upon by other systems.

> **Server-Side Flow (Outbound):**
> Game Logic (e.g., Fluid Simulation) -> Creates **FluidChange** instance -> `serialize()` -> Netty ByteBuf -> Network Transmission

> **Client-Side Flow (Inbound):**
> Network Reception -> Netty ByteBuf -> `deserialize()` -> Creates **FluidChange** instance -> World State Update -> Render Engine

