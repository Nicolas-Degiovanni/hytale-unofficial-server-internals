---
description: Architectural reference for DisconnectType
---

# DisconnectType

**Package:** com.hypixel.hytale.protocol.packets.connection
**Type:** Enumeration / Data Type

## Definition
```java
// Signature
public enum DisconnectType {
```

## Architecture & Concepts
The DisconnectType enum is a foundational component of the Hytale network protocol, responsible for providing a type-safe, standardized representation of disconnection reasons. It serves as a contract between the client and server, ensuring that the termination of a connection can be categorized and handled gracefully.

By mapping abstract concepts like a normal disconnect or a crash to fixed integer values, this enum eliminates the use of "magic numbers" within the packet serialization and deserialization logic. This is critical for protocol stability and forward compatibility. Its primary role is to populate a field within a higher-level disconnect packet, which is then interpreted by the connection handler to display an appropriate message or take specific recovery actions.

## Lifecycle & Ownership
- **Creation:** Instantiated by the Java Virtual Machine (JVM) during the initial class loading phase. The defined constants, Disconnect and Crash, are created once and only once.
- **Scope:** The enum constants are effectively singletons that persist for the entire application lifecycle.
- **Destruction:** Managed entirely by the JVM. These instances are not subject to conventional garbage collection and will be unloaded only when the class loader itself is garbage collected.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a final, private integer value that is assigned at compile time. The class itself is stateless.
- **Thread Safety:** Inherently thread-safe. As static final instances managed by the JVM, enum constants can be safely accessed and read from any thread without requiring external synchronization mechanisms.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the raw integer value associated with the enum constant, used for network serialization. |
| fromValue(int value) | DisconnectType | O(1) | **[Factory Method]** Deserializes an integer from a network stream into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is almost exclusively used during the serialization and deserialization of connection-related packets. The fromValue method is the designated entry point for converting raw network data into a type-safe object.

```java
// Deserializing a disconnect reason from a network buffer
int reasonCode = networkBuffer.readVarInt();
DisconnectType type = DisconnectType.fromValue(reasonCode);

// Acting on the deserialized type
if (type == DisconnectType.Crash) {
    Logger.error("The server reported a crash!");
    // Trigger crash reporting logic
}
```

### Anti-Patterns (Do NOT do this)
- **Reliance on ordinal():** Never use the built-in ordinal() method for serialization. The explicit value field guarantees protocol stability even if the declaration order of the enum constants changes in future versions. This class correctly avoids this pitfall.
- **Invalid Value Handling:** Do not wrap calls to fromValue in a generic try-catch block. The thrown ProtocolException is a specific, unrecoverable error indicating a corrupt or mismatched protocol version and should be handled at the highest level of the connection management stack.

## Data Pipeline
The DisconnectType enum acts as a translation point between the raw byte stream and the logical game state.

> Flow:
> Inbound TCP Stream -> Packet Deserializer -> **DisconnectType.fromValue(intValue)** -> S2CDisconnectPacket -> Connection State Machine -> UI Error Screen

