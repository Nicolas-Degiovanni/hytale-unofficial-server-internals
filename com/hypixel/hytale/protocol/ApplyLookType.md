---
description: Architectural reference for ApplyLookType
---

# ApplyLookType

**Package:** com.hypixel.hytale.protocol
**Type:** Enum Constant

## Definition
```java
// Signature
public enum ApplyLookType {
```

## Architecture & Concepts
ApplyLookType is a type-safe enumeration that serves as a discriminator within the Hytale network protocol. Its primary function is to define the specific interpretation or application method for player orientation data transmitted between the client and server.

This enum is a critical component of the protocol's serialization and deserialization layer. By mapping symbolic names like LocalPlayerLookOrientation to stable integer values, it eliminates the use of ambiguous "magic numbers" in the networking code. This ensures that protocol logic is both readable and robust against changes.

When a packet containing orientation data is received, the integer value is read from the byte stream and converted back into an ApplyLookType instance using the static fromValue factory method. This method includes strict validation, immediately rejecting any malformed packets with an invalid discriminator value by throwing a ProtocolException. This fail-fast behavior is essential for network security and stability.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated once by the Java Virtual Machine during the class-loading phase. They are compile-time constants, not runtime objects in the traditional sense.
- **Scope:** As static final instances, they exist for the entire lifetime of the application. Their scope is global.
- **Destruction:** The enum and its constants are unloaded from memory only when the JVM shuts down. There is no manual or garbage-collector-driven destruction.

## Internal State & Concurrency
- **State:** ApplyLookType is deeply immutable. Each enum constant is a singleton instance with a final, primitive integer value. The static VALUES array used for lookups is also final and populated only once at class-loading time.
- **Thread Safety:** This class is unconditionally thread-safe. Its immutability guarantees that no data races can occur. The static fromValue method is reentrant and safe for concurrent invocation from any thread, such as multiple network processing threads.

## API Surface
The public contract is minimal, focusing exclusively on serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the stable integer value for network serialization. |
| fromValue(int value) | ApplyLookType | O(1) | **Factory Method.** Deserializes an integer into an enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum should only be used during the process of reading from or writing to a network buffer. The fromValue method is the designated entry point for deserialization.

```java
// Example from a hypothetical packet deserializer
int lookTypeId = buffer.readVarInt();
ApplyLookType lookType = ApplyLookType.fromValue(lookTypeId);

// The game logic can now switch on the type-safe enum
switch (lookType) {
    case LocalPlayerLookOrientation:
        // handle specific orientation logic
        break;
    case Rotation:
        // handle generic rotation logic
        break;
}
```

### Anti-Patterns (Do NOT do this)
- **Using ordinal():** Do not use the built-in `ordinal()` method for serialization. The explicit `value` field is designed to be stable even if the declaration order of the enum constants changes. Relying on `ordinal()` will lead to protocol desynchronization.
- **Ignoring ProtocolException:** The fromValue method can throw a critical exception. This must be caught at the packet processing boundary to handle malicious or corrupted data. Failure to do so can crash the network thread.

## Data Pipeline
ApplyLookType acts as a deserialization gateway, converting a raw integer from the network into a validated, type-safe object for use in higher-level game systems.

> Flow:
> Network Byte Stream -> Protocol Deserializer -> **ApplyLookType.fromValue(readInt)** -> Game Logic Handler -> Player Entity Update

