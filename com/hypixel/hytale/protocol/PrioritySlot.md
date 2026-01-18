---
description: Architectural reference for PrioritySlot
---

# PrioritySlot

**Package:** com.hypixel.hytale.protocol
**Type:** Static Utility Enum

## Definition
```java
// Signature
public enum PrioritySlot {
```

## Architecture & Concepts
The PrioritySlot enum is a fundamental type within the network protocol layer. It provides a compile-time, type-safe representation for a fixed set of inventory or action slots, such as the main hand or off-hand. Its primary architectural purpose is to eliminate the use of "magic numbers" (e.g., raw integers 0, 1, 2) when serializing and deserializing game state over the network.

By mapping a human-readable name to a specific integer value, it enforces protocol correctness and improves code clarity. The static factory method, fromValue, acts as a validation and deserialization gateway. It ensures that any integer read from a network stream corresponds to a valid, known slot. If an invalid value is received, it immediately throws a ProtocolException, preventing data corruption from propagating further into the game state systems.

The static VALUES array is a performance optimization, caching the result of the expensive `values()` method call to allow for fast, allocation-free lookups during deserialization.

## Lifecycle & Ownership
- **Creation:** All enum constants (Default, MainHand, OffHand) are instantiated automatically by the Java Virtual Machine (JVM) when the PrioritySlot class is first loaded into memory. This process is managed entirely by the JVM class loader.
- **Scope:** Application-wide. As static final instances, these constants exist for the entire lifetime of the application and are shared globally.
- **Destruction:** The enum constants are garbage collected by the JVM only when the application is shutting down and its class loader is unloaded. No manual memory management is ever required.

## Internal State & Concurrency
- **State:** **Immutable**. Each PrioritySlot constant holds a private final integer value that is set at creation and can never be changed. The class itself has no other mutable state.
- **Thread Safety:** **Inherently thread-safe**. Due to its immutable nature, PrioritySlot can be safely accessed, passed, and read from any thread without requiring locks or other synchronization primitives. All methods are reentrant and free of side effects.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the raw integer value for network serialization. |
| fromValue(int value) | PrioritySlot | O(1) | **Deserialization Factory.** Converts an integer from a network stream into a PrioritySlot instance. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is used exclusively within the protocol layer for serializing and deserializing packets that reference specific equipment or action slots.

**Serialization (Writing to a packet):**
```java
// Game logic determines the slot to use
PrioritySlot slot = PrioritySlot.MainHand;

// The protocol writer gets the integer value for the network
int valueForNetwork = slot.getValue();
packetBuffer.writeInt(valueForNetwork);
```

**Deserialization (Reading from a packet):**
```java
// The protocol reader gets an integer from the network
int receivedValue = packetBuffer.readInt();

// The value is validated and converted back to a type-safe enum
try {
    PrioritySlot slot = PrioritySlot.fromValue(receivedValue);
    // ... process the packet with the valid slot
} catch (ProtocolException e) {
    // Handle corrupt or malicious packet data, typically by disconnecting the client
    log.error("Received invalid PrioritySlot value: " + receivedValue);
    connection.disconnect();
}
```

### Anti-Patterns (Do NOT do this)
- **Using Raw Integers:** Never use raw integers like `1` in game logic to represent a slot. This defeats the purpose of type safety and creates brittle code. Always use the enum constant, for example, `PrioritySlot.MainHand`.
- **Ignoring Deserialization Errors:** The `fromValue` method is a critical validation step. Failure to catch the ProtocolException it can throw will result in an unhandled exception that can crash the network thread or server. Always wrap calls to `fromValue` in a try-catch block within the packet decoding logic.

## Data Pipeline
PrioritySlot serves as a translation and validation component at the boundary between the raw network stream and the typed game state.

> **Serialization Flow:**
> Game State (PrioritySlot.MainHand) -> **getValue()** -> Integer (`1`) -> Network Buffer -> TCP Stream

> **Deserialization Flow:**
> TCP Stream -> Network Buffer -> Integer (`1`) -> **fromValue()** -> Game State (PrioritySlot.MainHand)

