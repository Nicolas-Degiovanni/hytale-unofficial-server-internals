---
description: Architectural reference for ClearEditorTimeOverride
---

# ClearEditorTimeOverride

**Package:** com.hypixel.hytale.protocol.packets.world
**Type:** Transient

## Definition
```java
// Signature
public class ClearEditorTimeOverride implements Packet {
```

## Architecture & Concepts
The ClearEditorTimeOverride packet is a stateless, zero-payload command within the Hytale network protocol. It serves a singular purpose: to signal the receiving endpoint to revert the in-game time to its natural cycle, disabling any previously set manual time override. This is typically used in creative or world editor modes.

Unlike data-carrying packets, this class acts as a pure signal. Its identity, defined by the static PACKET_ID 148, *is* the message. The protocol dispatcher identifies this ID and routes it to the appropriate handler, which then executes a predefined action without needing to parse any additional data from the packet body. This design is highly efficient for simple, unambiguous commands, as it minimizes network bandwidth and processing overhead.

This packet is a fundamental component of the client-server state synchronization model for world simulation parameters.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a system that needs to issue the command. For example, an editor UI service would create a new instance in response to a user clicking a "Reset Time" button.
- **Scope:** Extremely short-lived and transient. An instance exists only for the brief moment it is needed for serialization (on the sending side) or after deserialization before being passed to a handler (on the receiving side).
- **Destruction:** The object is immediately eligible for garbage collection after its corresponding handler has been invoked. It is not retained by any system.

## Internal State & Concurrency
- **State:** This object is **stateless and immutable**. It contains no fields and its behavior is constant. All instances of ClearEditorTimeOverride are functionally identical.
- **Thread Safety:** Inherently thread-safe. Due to its immutability, an instance can be safely shared and processed across any thread without locks or other synchronization primitives.

## API Surface
The public contract is dictated by the Packet interface. The implementation is trivial due to the packet's zero-payload nature.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the constant network identifier for this packet, 148. |
| serialize(ByteBuf) | void | O(1) | No-op. Writes zero bytes to the buffer as this packet has no payload. |
| deserialize(ByteBuf, int) | ClearEditorTimeOverride | O(1) | Static factory. Creates a new instance without reading from the buffer. |
| computeSize() | int | O(1) | Returns 0, indicating this packet has no data payload. |

## Integration Patterns

### Standard Usage
This packet should be created and immediately dispatched through the primary network service. It is a fire-and-forget command.

```java
// Example: A UI handler sending the command to the server.
// Do NOT hold a reference to the packet after sending.
networkManager.sendPacket(new ClearEditorTimeOverride());
```

### Anti-Patterns (Do NOT do this)
- **State Storage:** Do not attempt to add fields to this class or store an instance of it in another component. Its purpose is to be a transient event, not a state container. If data is required, a different packet type must be designed.
- **Redundant Sending:** Do not send this packet in a high-frequency loop. It should only be sent once in response to a specific user action or state change. The receiving system's state is binary (override active or not), so repeated commands are wasteful.

## Data Pipeline
The flow for this packet is simple, involving no data transformation beyond the identification of its type.

> **Outbound Flow:**
> User Action -> UI Event Handler -> `new ClearEditorTimeOverride()` -> Network Manager -> Packet Encoder (writes ID 148) -> TCP Stream

> **Inbound Flow:**
> TCP Stream -> Packet Decoder (reads ID 148) -> **Packet Dispatcher** -> `new ClearEditorTimeOverride()` -> World Time Packet Handler -> WorldTimeManager.clearOverride()

