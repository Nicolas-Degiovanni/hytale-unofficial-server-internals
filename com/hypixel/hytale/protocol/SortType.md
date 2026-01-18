---
description: Architectural reference for SortType
---

# SortType

**Package:** com.hypixel.hytale.protocol
**Type:** Static Enumeration / Value Type

## Definition
```java
// Signature
public enum SortType {
```

## Architecture & Concepts
The **SortType** enumeration represents a fixed set of type-safe constants used to define sorting criteria within the Hytale network protocol. Its primary function is to translate between a human-readable sorting option, such as **Name** or **Rarity**, and a compact integer representation suitable for network serialization.

This component is fundamental to the protocol layer, ensuring that both the client and server agree on a strict contract for data ordering requests. By mapping symbolic names to integer values, it minimizes payload size while eliminating the ambiguity and potential errors associated with using "magic numbers" directly in the application logic.

The inclusion of a custom **ProtocolException** for invalid values underscores its role in maintaining protocol integrity. An attempt to deserialize an undefined integer is treated as a protocol violation, allowing network handlers to reject malformed or malicious packets gracefully.

### Lifecycle & Ownership
- **Creation:** All **SortType** instances (**Name**, **Type**, **Rarity**) are created and initialized by the Java Virtual Machine during class loading. This process is automatic and occurs exactly once.
- **Scope:** The enum constants are static, final, and globally accessible. They persist for the entire lifetime of the application.
- **Destruction:** The instances are reclaimed by the JVM only when the application shuts down. There is no manual memory management or destruction process.

## Internal State & Concurrency
- **State:** **SortType** is deeply immutable. Each enum constant holds a final primitive integer, and the pre-cached **VALUES** array is also final. Its state cannot be modified after initialization.
- **Thread Safety:** This class is unconditionally thread-safe. As an immutable, JVM-managed structure, it can be safely accessed and used from any number of concurrent threads without requiring external synchronization or locks.

## API Surface
The public contract is designed for efficient serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer value for network serialization. |
| fromValue(int value) | static SortType | O(1) | Deserializes an integer into a SortType instance. Throws ProtocolException if the value is out of bounds. |
| VALUES | static SortType[] | O(1) | A cached, public array of all enum constants. This avoids the performance overhead of calling the internal values() method repeatedly. |

## Integration Patterns

### Standard Usage
The primary use case is within network packet handlers to serialize or deserialize data related to list sorting.

```java
// Example: Deserializing a sort type from a network buffer
int sortTypeId = networkBuffer.readVarInt();
try {
    SortType sortOrder = SortType.fromValue(sortTypeId);
    // Now use sortOrder to sort a list of items
    itemList.sort(getComparatorFor(sortOrder));
} catch (ProtocolException e) {
    // Handle malformed packet, perhaps by disconnecting the client
    log.error("Received invalid SortType ID: " + sortTypeId);
}
```

### Anti-Patterns (Do NOT do this)
- **Using Magic Numbers:** Avoid using raw integers (0, 1, 2) in game logic. This creates brittle code that is difficult to read and maintain. Always reference the enum constants directly, such as **SortType.Rarity**.
- **Ignoring Exceptions:** Failure to catch **ProtocolException** from the **fromValue** method can lead to unhandled exceptions in the network processing pipeline, potentially crashing a session. Always wrap calls in a try-catch block when processing external data.
- **Using ordinal():** Do not use the built-in **ordinal()** method for serialization. The integer values are explicitly defined and guaranteed to be stable, whereas the ordinal can change if the declaration order is modified. Always use **getValue()**.

## Data Pipeline
**SortType** acts as a translation layer at the boundary between the application logic and the network serialization stream.

**Serialization Flow (Outgoing)**
> Flow:
> Game Logic Request (e.g., **SortType.Name**) -> **getValue()** -> Integer (0) -> Network Packet Encoder -> TCP Stream

**Deserialization Flow (Incoming)**
> Flow:
> TCP Stream -> Network Packet Decoder -> Integer (e.g., 2) -> **fromValue(2)** -> **SortType.Rarity** -> Game Logic Handler

