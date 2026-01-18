---
description: Architectural reference for CustomPageEventType
---

# CustomPageEventType

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Enumeration / Utility

## Definition
```java
// Signature
public enum CustomPageEventType {
```

## Architecture & Concepts
CustomPageEventType provides a type-safe, compile-time constant representation for events related to the server-driven custom user interface system. Its primary architectural function is to act as a strict contract between the client and server for serializing and deserializing UI event identifiers.

This enum is a critical component of the network protocol layer. It translates raw integer values, which are efficient for network transmission, into meaningful, self-documenting constants used throughout the game's business logic. This pattern eliminates the use of "magic numbers" in the application code, significantly improving readability and reducing the risk of errors when handling different UI events.

The inclusion of the `fromValue` factory method, which throws a ProtocolException on invalid input, establishes a validation gateway. Any malformed or unsupported event type value received from the network is immediately rejected at the deserialization stage, preventing invalid state from propagating deeper into the application.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine during class loading. They are effectively static singletons created once when the application starts.
- **Scope:** Application-wide. These constants persist for the entire lifetime of the JVM process.
- **Destruction:** The constants are garbage collected only when the class loader itself is unloaded, which typically occurs at application shutdown. There is no manual destruction or cleanup required.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a final integer `value` that is set at creation and can never be changed. The `VALUES` array is also static and final.
- **Thread Safety:** This enum is inherently thread-safe. As immutable singletons, its constants can be safely accessed, passed, and compared across any number of threads without locks or other synchronization primitives. The static factory method `fromValue` is also fully thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the raw integer value associated with the enum constant, used for network serialization. |
| fromValue(int value) | CustomPageEventType | O(1) | **[Factory]** Deserializes an integer into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during packet serialization and deserialization. The `fromValue` method decodes the network value, and `getValue` encodes the application value.

```java
// Deserializing an incoming packet
int eventTypeId = buffer.readVarInt();
CustomPageEventType eventType = CustomPageEventType.fromValue(eventTypeId);

switch (eventType) {
    case Data:
        // Handle incoming UI data
        break;
    case Dismiss:
        // Close the UI page
        break;
    // ...
}

// Serializing an outgoing packet
CustomPageEventType eventToSend = CustomPageEventType.Acknowledge;
buffer.writeVarInt(eventToSend.getValue());
```

### Anti-Patterns (Do NOT do this)
- **Comparison by Value:** Never compare enum instances by their underlying integer value. This defeats the purpose of type safety and makes the code brittle.

    ```java
    // BAD: Prone to errors and hard to read
    if (eventType.getValue() == 2) { /* ... */ }

    // GOOD: Type-safe and self-documenting
    if (eventType == CustomPageEventType.Dismiss) { /* ... */ }
    ```

- **Ignoring Deserialization Errors:** The ProtocolException thrown by `fromValue` is a critical signal of a protocol violation or data corruption. It must not be ignored. Swallowing this exception can lead to unpredictable application state. The offending connection should typically be terminated.

    ```java
    // DANGEROUS: Hides a critical protocol error
    try {
        eventType = CustomPageEventType.fromValue(id);
    } catch (ProtocolException e) {
        // Silently ignoring the error
        eventType = null;
    }
    ```

## Data Pipeline
CustomPageEventType serves as the translation point between the raw network stream and the typed application logic for UI events.

> Flow:
> Network Byte Stream -> Packet Deserializer -> `CustomPageEventType.fromValue(readInt())` -> **CustomPageEventType Instance** -> Packet Handler / Game Logic

