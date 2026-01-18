---
description: Architectural reference for ChangeBlockAction
---

# ChangeBlockAction

**Package:** com.hypixel.hytale.protocol.packets.window
**Type:** Transient

## Definition
```java
// Signature
public class ChangeBlockAction extends WindowAction {
```

## Architecture & Concepts
The ChangeBlockAction class is a specialized Data Transfer Object (DTO) within the Hytale network protocol. It inherits from WindowAction, indicating it represents a user-driven event originating from a UI window. Its specific purpose is to communicate a binary state change for a UI component, such as a toggle switch or checkbox being activated or deactivated.

This class is designed for extreme efficiency and low network overhead. It serializes to a single byte, making it ideal for high-frequency UI interactions where performance is critical. It serves as a fundamental building block for synchronizing client-side UI state with server-side game logic. The existence of static deserialization and validation methods is characteristic of the protocol's packet factory system, which decodes raw byte streams into concrete action objects.

### Lifecycle & Ownership
- **Creation:** An instance is created on the client when a user interacts with a corresponding UI element. On the receiving end (client or server), it is instantiated by a protocol deserializer when a packet of this type is read from the network buffer.
- **Scope:** The object's lifetime is exceptionally short. It exists only for the brief period required to be serialized, transmitted, deserialized, and processed by a handler. It is a "fire-and-forget" message.
- **Destruction:** The object is immediately eligible for garbage collection after its state has been consumed by the relevant game or UI logic handler. No system maintains a long-term reference to it.

## Internal State & Concurrency
- **State:** The internal state consists of a single mutable boolean field, *down*. This field directly represents the payload of the packet. The class holds no other cached, derived, or complex state.
- **Thread Safety:** This class is **not thread-safe**. It is a plain data container with no internal synchronization mechanisms. It is intended to be created, handled, and discarded within the confines of a single thread, typically a Netty I/O thread or the main game update loop. Sharing instances across threads without external locking is an anti-pattern and will lead to data corruption.

## API Surface
The public API is focused entirely on serialization, deserialization, and state access, consistent with its role as a network packet.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf buf) | int | O(1) | Encodes the object's state into the provided Netty ByteBuf. Returns the number of bytes written (always 1). |
| deserialize(ByteBuf buf, int offset) | ChangeBlockAction | O(1) | **Static Factory.** Creates a new ChangeBlockAction instance by reading from the buffer at the given offset. |
| computeSize() | int | O(1) | Returns the fixed size of the packet in bytes (always 1). |
| validateStructure(ByteBuf buffer, int offset) | ValidationResult | O(1) | **Static.** Performs a pre-check to ensure the buffer contains enough data to deserialize the object. |
| clone() | ChangeBlockAction | O(1) | Creates a shallow copy of the object. |

## Integration Patterns

### Standard Usage
This object is almost never instantiated directly by game logic developers. Instead, it is produced by the network layer's deserialization pipeline and consumed by a registered handler.

```java
// Example of a handler processing the action
void handleWindowAction(WindowAction action) {
    if (action instanceof ChangeBlockAction cba) {
        // Logic to update game state based on the toggle
        boolean isPressed = cba.down;
        this.updateComponentState(windowId, componentId, isPressed);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not retain an instance of ChangeBlockAction to be modified and re-sent later. Each distinct user action must result in the creation of a new instance to ensure state integrity.
- **Cross-Thread Sharing:** Do not pass an instance from the network thread to a game logic thread without a proper synchronization boundary. It is safer to extract the primitive *down* value and pass that instead.

## Data Pipeline
The flow of data for this object is linear and unidirectional, from user input to state change.

> **Client to Server Flow:**
> UI Interaction (Checkbox Click) -> **new ChangeBlockAction(true)** -> Protocol Serializer -> Netty ByteBuf -> Network -> Server Deserializer -> **ChangeBlockAction Instance** -> Game Logic Handler -> World State Update

