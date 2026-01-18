---
description: Architectural reference for CustomHud
---

# CustomHud

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Transient

## Definition
```java
// Signature
public class CustomHud implements Packet {
```

## Architecture & Concepts
The CustomHud class is a network Packet, a specialized Data Transfer Object (DTO) designed to transmit user interface modification commands from the server to the client. Its sole purpose is to act as a data contract for serializing and deserializing instructions that dynamically alter the player's Heads-Up Display (HUD).

This packet is a fundamental component of the server-authoritative UI system. Rather than having the client manage its own HUD state, the server sends CustomHud packets to instruct the client to render, update, or remove specific UI elements. This allows for dynamic, context-sensitive user interfaces controlled by server-side game logic, such as quest objectives, boss health bars, or temporary status effects.

It does not contain any logic itself; it is a pure data container processed by the client's network and UI rendering pipeline.

## Lifecycle & Ownership
- **Creation:**
    - **Outbound (Server):** Instantiated by server-side game systems when a player's HUD needs to be updated. It is populated with a series of CustomUICommand objects and then passed to the network layer for serialization.
    - **Inbound (Client):** Instantiated by the client's protocol deserializer when a network buffer with Packet ID 217 is received. The static deserialize method is the designated factory.

- **Scope:** Extremely short-lived. A CustomHud object exists only for the brief moment between deserialization and processing by a packet handler. It is not intended to be stored or referenced long-term.

- **Destruction:** The object is eligible for garbage collection immediately after the client-side packet handler has consumed its data and applied the changes to the active UI state.

## Internal State & Concurrency
- **State:** Mutable. The class is a simple data structure with public fields. Its primary state consists of a boolean flag, clear, and a nullable array of CustomUICommand objects. The clear flag indicates whether the client should discard all existing custom HUD elements before applying the new commands.

- **Thread Safety:** **Not thread-safe.** This class is a simple POJO with no internal locking or synchronization. It is designed to be created, serialized, deserialized, and processed by a single thread, typically a Netty network thread or a main game-loop thread.

    **WARNING:** Concurrent modification of a CustomHud instance from multiple threads will lead to race conditions and undefined behavior. Any handoff between threads, such as from a network thread to a UI thread, must be managed with external synchronization or a thread-safe queue.

## API Surface
The public API is designed for use by the protocol framework, not general application code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static CustomHud | O(N) | Constructs a CustomHud object by reading from a ByteBuf. N is the number of commands. |
| serialize(buf) | void | O(N) | Writes the object's state into a ByteBuf for network transmission. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the object. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a read-only check of a buffer to ensure it contains a valid CustomHud structure. |
| clone() | CustomHud | O(N) | Creates a deep copy of the packet and its contained commands. |

## Integration Patterns

### Standard Usage
The class is intended to be used by the network protocol layer. A packet handler receives the deserialized object and dispatches its contents to the appropriate UI management system.

```java
// On the client, within a packet handler
public void handleCustomHud(CustomHud packet) {
    UIManager uiManager = context.getService(UIManager.class);

    if (packet.clear) {
        uiManager.clearAllCustomElements();
    }

    if (packet.commands != null) {
        for (CustomUICommand command : packet.commands) {
            uiManager.processCommand(command);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not store instances of CustomHud in caches or as component state. Extract the data you need and discard the packet object. Its purpose is to transmit, not to store.

- **Manual Serialization:** Avoid calling serialize or deserialize directly in game logic. These methods are the responsibility of the network protocol engine. Game code should create the object and pass it to a network dispatch service.

- **Reusing Instances:** Do not modify and resend the same CustomHud instance. Packets are cheap to create and should be treated as immutable once created for sending.

## Data Pipeline
The CustomHud packet is a critical link in the server-to-client UI update pipeline.

> **Outbound Flow (Server):**
> Game Event -> UI Logic Creates **CustomHud** -> Protocol Encoder calls `serialize` -> Netty ByteBuf -> TCP/IP Stack

> **Inbound Flow (Client):**
> TCP/IP Stack -> Netty ByteBuf -> Protocol Decoder identifies ID 217 -> `CustomHud.deserialize` creates **CustomHud** -> Packet Handler -> UIManager -> Render Engine Update

