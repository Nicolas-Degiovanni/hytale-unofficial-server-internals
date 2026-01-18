---
description: Architectural reference for CustomUIEventBindingType
---

# CustomUIEventBindingType

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Static Enumeration

## Definition
```java
// Signature
public enum CustomUIEventBindingType {
```

## Architecture & Concepts
CustomUIEventBindingType is a static enumeration that defines a strict contract for all possible user interface interaction events that can be communicated between the client and server. It serves as a foundational component of the custom UI networking protocol.

By mapping specific user actions (e.g., a right-click, a key press) to fixed integer values, this enum eliminates the use of ambiguous "magic numbers" in the network stream. This ensures that both the client-side UI framework and the server-side game logic interpret user interactions identically.

The primary architectural role of this enum is to facilitate the serialization and deserialization of UI events. The client serializes an enum member to its integer representation for network transmission, and the server deserializes the integer back into the corresponding enum member for processing. The static `fromValue` method acts as the primary deserialization factory, providing critical input validation against the known set of event types.

## Lifecycle & Ownership
- **Creation:** All instances of this enum are created and initialized by the Java Virtual Machine during class loading. They are compile-time constants and cannot be instantiated at runtime.
- **Scope:** Application-wide. The enum constants exist for the entire lifetime of the application once the class is loaded.
- **Destruction:** The enum and its instances are unloaded only when the application's class loader is garbage collected, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a final, private integer `value`. The static `VALUES` array, used for efficient lookups, is also effectively immutable after its one-time static initialization.
- **Thread Safety:** This class is inherently thread-safe. As an immutable, globally accessible set of constants, it can be safely read from any thread without requiring synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer value assigned to the enum constant for network serialization. |
| fromValue(int value) | CustomUIEventBindingType | O(1) | Deserializes an integer from the network into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is used during the serialization and deserialization of network packets containing UI data.

**Serialization (Client-Side)**
```java
// When sending a UI event to the server, get its integer value.
int networkValue = CustomUIEventBindingType.KeyDown.getValue();
packet.writeInt(networkValue);
```

**Deserialization (Server-Side)**
```java
// When receiving a packet, convert the integer back to a type-safe enum.
int receivedValue = packet.readInt();
try {
    CustomUIEventBindingType eventType = CustomUIEventBindingType.fromValue(receivedValue);
    // Use a switch for clear, type-safe event handling.
    switch (eventType) {
        case KeyDown:
            // handle key down logic
            break;
        case FocusLost:
            // handle focus lost logic
            break;
        // ... other cases
    }
} catch (ProtocolException e) {
    // Handle malicious or malformed packet data.
    // Disconnect client or log warning.
}
```

### Anti-Patterns (Do NOT do this)
- **Using Ordinals:** Do not use the built-in `ordinal()` method for serialization. The integer values are explicitly defined and decoupled from declaration order. Relying on `ordinal()` is fragile and will break the protocol if a new enum constant is inserted in the middle of the list. Always use `getValue()`.
- **Unsafe Deserialization:** Do not attempt to deserialize by accessing the `VALUES` array directly without bounds checking. The static `fromValue` method is the designated entry point for deserialization as it provides essential validation.
- **Ignoring ProtocolException:** Failure to catch the ProtocolException thrown by `fromValue` can lead to unhandled exceptions and server instability when processing invalid data from a client.

## Data Pipeline
This enum is a critical translation layer in the UI event data pipeline, converting high-level UI objects into a low-level network-safe format and back again.

> **Client Flow:**
> User Input -> UI Component Event -> **CustomUIEventBindingType** -> `getValue()` -> Integer in Network Packet

> **Server Flow:**
> Integer in Network Packet -> `fromValue(int)` -> **CustomUIEventBindingType** -> Game Logic Event Handler

