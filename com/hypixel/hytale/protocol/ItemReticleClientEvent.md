---
description: Architectural reference for ItemReticleClientEvent
---

# ItemReticleClientEvent

**Package:** com.hypixel.hytale.protocol
**Type:** Utility

## Definition
```java
// Signature
public enum ItemReticleClientEvent {
```

## Architecture & Concepts
ItemReticleClientEvent is a type-safe enumeration that defines a fixed set of states or actions related to the player's targeting reticle. It serves as a contract within the client-server communication protocol, translating high-level gameplay events into a compact, integer-based format suitable for network transmission.

The primary architectural function of this enum is to eliminate "magic numbers" from the networking layer. By using symbolic names like OnHit instead of the raw integer 0, the protocol becomes more readable, maintainable, and less prone to errors. The class provides bidirectional conversion: from the symbolic enum to its integer value for serialization (via getValue) and from a received integer back to the symbolic enum for deserialization (via fromValue).

This component is fundamental to the client-side input and combat systems, acting as the data payload for packets that inform the server about player interactions with the world via their reticle.

### Lifecycle & Ownership
- **Creation:** All instances of this enum (OnHit, Wielding, etc.) are created and initialized by the Java Virtual Machine during class loading. They are compile-time constants and exist as singletons managed by the JVM.
- **Scope:** Application-wide. Once the ItemReticleClientEvent class is loaded, these instances persist for the entire lifetime of the application.
- **Destruction:** The enum instances are reclaimed by the garbage collector only when the application's class loader is unloaded, which typically occurs at shutdown. Direct manual destruction is not possible.

## Internal State & Concurrency
- **State:** **Immutable**. Each enum constant has a final `value` field that is set at creation and can never be changed. The state of an ItemReticleClientEvent is fixed for the application's lifetime.
- **Thread Safety:** **Inherently thread-safe**. Due to their immutable and singleton nature, these enum constants can be safely accessed, passed, and read from any thread without requiring external synchronization or locks.

## API Surface
The public contract is designed for serialization and deserialization of network data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the low-level integer value associated with the enum constant, intended for network serialization. |
| fromValue(int value) | ItemReticleClientEvent | O(1) | A static factory method that converts a raw integer from a network packet back into its corresponding enum constant. Throws ProtocolException if the integer is out of the defined range. |

## Integration Patterns

### Standard Usage
This enum is used when serializing client input for the server or deserializing server-bound packets.

```java
// Example: Client sending an event to the server
// The client's input system detects a successful hit.
ItemReticleClientEvent eventToSend = ItemReticleClientEvent.OnHit;
int valueForNetwork = eventToSend.getValue();
// networkBuffer.writeInt(valueForNetwork);


// Example: Server processing a received event
// int receivedValue = networkBuffer.readInt();
int receivedValue = 0; // Simulating a received value of 0
try {
    ItemReticleClientEvent receivedEvent = ItemReticleClientEvent.fromValue(receivedValue);
    if (receivedEvent == ItemReticleClientEvent.OnHit) {
        // Process combat logic for a successful hit...
    }
} catch (ProtocolException e) {
    // Handle malicious or corrupted packet data.
}
```

### Anti-Patterns (Do NOT do this)
- **Using Raw Integers:** Avoid using the integer values directly in game logic. Comparing a variable to 0 is less clear and more brittle than comparing it to ItemReticleClientEvent.OnHit. Relying on the integer value breaks encapsulation and makes the code harder to refactor if the protocol changes.
- **Ignoring Exceptions:** The fromValue method can throw a ProtocolException. Failure to catch this exception can lead to server crashes or unhandled states when processing malformed or malicious client packets. Always wrap calls to fromValue in a try-catch block at the network boundary.

## Data Pipeline
ItemReticleClientEvent acts as a data payload that flows from the client's game logic to the server's game logic via the network protocol.

> Flow:
> Client Input (e.g., Mouse Movement, Attack) -> Game Logic -> **ItemReticleClientEvent** -> Protocol Serializer -> Network Packet -> Server -> Protocol Deserializer -> Server-Side Game Logic

