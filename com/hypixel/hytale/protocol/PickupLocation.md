---
description: Architectural reference for PickupLocation
---

# PickupLocation

**Package:** com.hypixel.hytale.protocol
**Type:** Static Enum

## Definition
```java
// Signature
public enum PickupLocation {
```

## Architecture & Concepts
PickupLocation is a type-safe enumeration that represents a fixed set of inventory destinations within the game world. It serves as a critical component of the network protocol layer, translating between human-readable game logic concepts (*Hotbar*, *Storage*) and their compact, integer-based representations for network serialization.

The primary architectural function of this enum is to enforce data integrity and prevent the use of ambiguous "magic numbers" in the game's inventory and item management systems. By providing a constrained set of valid locations, it ensures that both the client and server agree on where an item is being sent.

The static factory method, fromValue, acts as a validation gateway during packet deserialization. Any attempt to construct a PickupLocation from an undefined integer value will result in a ProtocolException, immediately halting the processing of the invalid packet and preventing state corruption.

## Lifecycle & Ownership
- **Creation:** Enum constants are instantiated automatically by the Java Virtual Machine (JVM) during the initial class loading phase. This process is managed entirely by the JVM and occurs once per application lifecycle.
- **Scope:** Application-wide. The instances (Hotbar, Storage) are static, final, and globally accessible, persisting for the entire duration of the application.
- **Destruction:** The enum constants are reclaimed by the garbage collector only when the application's ClassLoader is unloaded, which typically happens at JVM shutdown.

## Internal State & Concurrency
- **State:** Deeply immutable. Each enum constant holds a private final integer, which is assigned at creation and can never be modified. The static VALUES array is also final and populated at class-load time, preventing runtime modification.
- **Thread Safety:** Inherently thread-safe. Due to its complete immutability, PickupLocation can be safely read and passed between any number of threads without requiring locks or other synchronization mechanisms.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the low-level integer value for network serialization. |
| fromValue(int value) | static PickupLocation | O(1) | Deserializes an integer into a PickupLocation instance. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used when serializing or deserializing network packets related to item and inventory interactions.

```java
// Example: Deserializing an incoming packet
int locationId = packet.readVarInt();
try {
    PickupLocation destination = PickupLocation.fromValue(locationId);
    player.getInventory().placeItem(item, destination);
} catch (ProtocolException e) {
    // Handle corrupted or malicious packet
    connection.disconnect("Invalid pickup location in packet.");
}

// Example: Serializing an outgoing packet
PickupLocation source = player.getHotbarLocation();
packet.writeVarInt(source.getValue());
```

### Anti-Patterns (Do NOT do this)
- **Integer Comparison:** Do not use the integer value for logical comparisons. Rely on the type-safe enum constants directly. This practice is brittle and circumvents the safety provided by the enum.
  ```java
  // BAD
  if (location.getValue() == 0) { /* ... */ }

  // GOOD
  if (location == PickupLocation.Hotbar) { /* ... */ }
  ```
- **Unchecked Deserialization:** Never call fromValue without a try-catch block when processing external network data. An unhandled ProtocolException can crash the packet processing thread.

## Data Pipeline
PickupLocation acts as a translation and validation point in the data flow between the network and game logic layers.

> **Incoming Data Flow (Deserialization):**
> Raw Byte Stream -> Protocol Decoder (reads integer) -> **PickupLocation.fromValue(int)** -> Game Logic (uses PickupLocation object)

> **Outgoing Data Flow (Serialization):**
> Game Logic (uses PickupLocation object) -> **pickupLocation.getValue()** -> Protocol Encoder (writes integer) -> Raw Byte Stream

