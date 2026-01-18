---
description: Architectural reference for EditorSelection
---

# EditorSelection

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Transient Data Object

## Definition
```java
// Signature
public class EditorSelection {
```

## Architecture & Concepts
The EditorSelection class is a Data Transfer Object (DTO) that represents a fixed-size, axis-aligned bounding box (AABB) within the Hytale network protocol. Its primary role is to provide a standardized binary representation for a 3D volumetric selection, typically used by in-game editing tools.

This class is not a general-purpose AABB for physics or rendering calculations. It is specifically designed for network serialization and deserialization, acting as a data contract between the client and server. Its structure is intentionally rigid, with a fixed block size of 24 bytes, to ensure predictable and high-performance processing on the Netty-based network layer. The use of Little Endian byte order in its serialization methods is a core part of this protocol contract.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary circumstances:
    1. By a network packet deserializer, which calls the static `deserialize` method to construct an object from an incoming Netty `ByteBuf`.
    2. By game logic, typically within the editor systems, when a user action creates a new selection that must be transmitted to the server or other clients.
- **Scope:** The lifecycle of an EditorSelection object is extremely short. It is a transient object, intended to exist only for the duration of a single operation, such as processing one network packet or handling one UI event. It is passed by value or cloned, not held as a long-lived reference.
- **Destruction:** Instances are managed by the Java Garbage Collector and are eligible for cleanup as soon as they are no longer referenced. There are no manual resource management or destruction methods.

## Internal State & Concurrency
- **State:** The state is entirely mutable. All six integer fields representing the bounding box coordinates are public and can be modified directly at any time. The object acts as a simple data container with no internal logic to protect its state.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, manipulated, and read within a single thread, such as a Netty I/O thread or the main game logic thread. Concurrent modification from multiple threads will lead to race conditions and data corruption. All synchronization must be handled externally by the calling system.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EditorSelection(int...) | constructor | O(1) | Creates a new instance with specified coordinates. |
| deserialize(ByteBuf, int) | static EditorSelection | O(1) | Constructs an object by reading 24 bytes from a buffer. |
| serialize(ByteBuf) | void | O(1) | Writes the object's state as 24 bytes into a buffer. |
| computeSize() | int | O(1) | Returns the constant size of the object's binary form (24). |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Checks if a buffer has enough readable bytes to contain this structure. |
| clone() | EditorSelection | O(1) | Creates a deep copy of the object. |

## Integration Patterns

### Standard Usage
EditorSelection is almost always used as a component of a larger network packet. The parent packet is responsible for invoking the serialization or deserialization logic.

```java
// Example: Deserializing from a network buffer inside a parent packet
public void deserialize(ByteBuf buf) {
    // ... deserialize other packet fields ...
    int selectionOffset = ...; // Calculate offset
    this.selection = EditorSelection.deserialize(buf, selectionOffset);
    // ... continue deserializing ...
}

// Example: Creating and serializing a new selection
EditorSelection newSelection = new EditorSelection(0, 0, 0, 16, 16, 16);
SomeEditorPacket packet = new SomeEditorPacket(newSelection);

// The packet's serialize method would then call:
// newSelection.serialize(outputByteBuf);
```

### Anti-Patterns (Do NOT do this)
- **Long-Term State:** Do not store instances of EditorSelection as long-term state in game systems. Its mutable nature makes it unsuitable for this. Convert it to an immutable or engine-specific AABB class for use in game logic.
- **Cross-Thread Sharing:** Never share an instance of this class across threads without explicit and robust locking. It is fundamentally unsafe for concurrent access.
- **Misuse for Game Logic:** Avoid using this class for physics queries, collision detection, or rendering bounds. The engine will have dedicated, more performant classes (e.g., with vector math libraries) for those tasks. This class is for network I/O only.

## Data Pipeline
The class serves as a translation point between raw network bytes and a structured in-memory representation of a 3D volume.

> **Incoming Flow:**
> Netty ByteBuf -> Parent Packet Deserializer -> **EditorSelection.deserialize** -> Game Logic (Editor System)

> **Outgoing Flow:**
> User Action -> Game Logic (Editor System) -> **new EditorSelection()** -> Parent Packet Serializer -> **EditorSelection.serialize** -> Netty ByteBuf

