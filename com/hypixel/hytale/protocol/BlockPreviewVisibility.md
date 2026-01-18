---
description: Architectural reference for BlockPreviewVisibility
---

# BlockPreviewVisibility

**Package:** com.hypixel.hytale.protocol
**Type:** Utility / Data Type

## Definition
```java
// Signature
public enum BlockPreviewVisibility {
```

## Architecture & Concepts
BlockPreviewVisibility is a protocol-level enumeration that defines a fixed set of states for rendering the client-side block placement preview. It serves as a type-safe contract between the client and server, ensuring that settings related to this UI feature are communicated without ambiguity.

Its primary architectural function is to serialize a high-level game concept (visibility state) into a low-level, network-efficient integer. This pattern is common throughout the Hytale protocol stack to minimize packet size and processing overhead. The enum encapsulates the mapping between the symbolic names (AlwaysVisible, AlwaysHidden, Default) and their corresponding integer wire format values.

The inclusion of a static factory method, fromValue, provides a single, robust entry point for deserialization. This method acts as a validation gate, throwing a ProtocolException for any out-of-bounds integer, which prevents the propagation of corrupt or malicious data into the game state.

### Lifecycle & Ownership
- **Creation:** Instances are created and managed exclusively by the Java Virtual Machine during class loading. As an enum, its constants are singleton instances that are initialized once.
- **Scope:** Application-wide. The enum constants persist for the entire lifetime of the client or server process.
- **Destruction:** Instances are garbage collected only when the JVM shuts down. There is no manual memory management.

## Internal State & Concurrency
- **State:** Immutable. The internal integer value for each enum constant is declared final and assigned at compile time. The state of a BlockPreviewVisibility instance can never change after its creation.
- **Thread Safety:** Inherently thread-safe. Due to its immutability and the JVM's guarantees for enum initialization, this class can be safely accessed and shared across any number of threads without external synchronization or locking mechanisms.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer representation of the enum constant for network serialization. |
| fromValue(int value) | BlockPreviewVisibility | O(1) | **Static.** Deserializes an integer from a network packet into its corresponding enum constant. Throws ProtocolException if the value is invalid. |

## Integration Patterns

### Standard Usage
This enum is primarily used during the serialization and deserialization of network packets or game settings.

**Serialization (Writing to a packet):**
```java
// Get the integer value to send over the network
int valueToSend = BlockPreviewVisibility.AlwaysVisible.getValue();
packetBuffer.writeInt(valueToSend);
```

**Deserialization (Reading from a packet):**
```java
// Read the integer and convert it back to the type-safe enum
int receivedValue = packetBuffer.readInt();
BlockPreviewVisibility setting = BlockPreviewVisibility.fromValue(receivedValue);
// Now use the 'setting' variable in game logic
```

### Anti-Patterns (Do NOT do this)
- **Using Magic Numbers:** Never use the raw integer values (0, 1, 2) directly in game logic. This defeats the purpose of type safety and makes the code difficult to read and maintain. Always reference the symbolic constants.

    ```java
    // BAD: Hard to understand, brittle if protocol changes
    if (player.getPreviewSetting() == 0) {
        // ...
    }

    // GOOD: Clear, type-safe, and refactor-friendly
    if (player.getPreviewSetting() == BlockPreviewVisibility.AlwaysVisible) {
        // ...
    }
    ```
- **Ignoring ProtocolException:** The fromValue method can throw an exception. Failure to handle this can lead to a client or server crash if invalid data is received. Always deserialize within a try-catch block at the protocol boundary.

## Data Pipeline
BlockPreviewVisibility acts as a data model, not an active processing component. It represents data at rest within the pipeline.

> **Serialization Flow:**
> Game Setting (BlockPreviewVisibility.Default) -> **getValue()** -> Integer (2) -> Network Packet Buffer -> TCP/IP Stack

> **Deserialization Flow:**
> TCP/IP Stack -> Network Packet Buffer -> Integer (e.g., 0) -> **fromValue(0)** -> Game Setting (BlockPreviewVisibility.AlwaysVisible)

