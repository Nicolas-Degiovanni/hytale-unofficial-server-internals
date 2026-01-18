---
description: Architectural reference for ClickType
---

# ClickType

**Package:** com.hypixel.hytale.protocol
**Type:** Utility

## Definition
```java
// Signature
public enum ClickType {
```

## Architecture & Concepts
The ClickType enum is a foundational data type within the Hytale network protocol, responsible for providing a type-safe representation of discrete mouse button actions. While simple in structure, it serves a critical architectural role as a data contract between the client and server.

Its primary function is to translate low-level integer identifiers, which are efficient for network transmission, into high-level, self-documenting constants for use within the game logic. This translation is bidirectional, ensuring that both serialization (game state to network packet) and deserialization (network packet to game state) are consistent and robust.

The inclusion of the static factory method, fromValue, elevates this enum from a simple container to a protocol validation component. By performing strict bounds checking, it acts as a gatekeeper, preventing invalid or potentially malicious integer values from propagating into the game engine and causing undefined behavior. A failure at this stage signifies a protocol violation and is handled by throwing a ProtocolException.

## Lifecycle & Ownership
- **Creation:** As a Java enum, all instances (None, Left, Right, Middle) are constructed and initialized by the JVM during class loading. They are compile-time constants and exist as singletons managed by the runtime.
- **Scope:** Application-level. The ClickType constants are available for the entire lifetime of the application once the ClickType class is loaded.
- **Destruction:** The enum instances are reclaimed by the JVM only when the defining class loader is garbage collected, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** Deeply immutable. Each enum constant holds a final primitive integer. Its state cannot be modified after creation. The static VALUES array is also final and its contents are immutable.
- **Thread Safety:** Inherently thread-safe. Due to its complete immutability, ClickType can be safely accessed, read, and passed between any number of threads without requiring external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the raw integer representation of the enum constant for network serialization. |
| fromValue(int value) | ClickType | O(1) | **Critical:** Deserializes an integer from a network stream into a ClickType instance. Throws ProtocolException if the value is out of the defined range. |

## Integration Patterns

### Standard Usage
This component is primarily used within packet serialization and deserialization logic. The pattern is to convert to and from the raw integer value at the protocol boundary.

```java
// Example: Deserializing an incoming player action packet
int rawClickType = networkBuffer.readVarInt();
ClickType action = ClickType.fromValue(rawClickType); // Validation occurs here

// Example: Serializing an outgoing packet
PlayerActionPacket packet = new PlayerActionPacket();
packet.setClick(ClickType.Left.getValue());
```

### Anti-Patterns (Do NOT do this)
- **Magic Numbers:** Avoid using raw integer values (0, 1, 2, 3) directly in game logic. This creates brittle code that is difficult to read and maintain. Always reference the enum constants, for example, ClickType.Left.
- **Swallowing Exceptions:** Never wrap a call to fromValue in a generic try-catch block that ignores the ProtocolException. This exception signals a malformed packet or a client/server version mismatch, which is a critical error that must be handled, often by terminating the connection.
- **Manual Lookups:** Do not iterate over `ClickType.values()` to find a match for an integer. The `fromValue(int)` method is a highly optimized and safer alternative that uses a direct array lookup.

## Data Pipeline
ClickType acts as a translation and validation point in the data flow between the network layer and the game logic.

> **Inbound Flow (Deserialization):**
> Raw Integer (from Network Packet) -> **ClickType.fromValue()** -> Type-Safe Enum (in Game Logic)

> **Outbound Flow (Serialization):**
> Type-Safe Enum (in Game Logic) -> **ClickType.getValue()** -> Raw Integer (for Network Packet)

