---
description: Architectural reference for the SwitchTo protocol enumeration.
---

# SwitchTo

**Package:** com.hypixel.hytale.protocol
**Type:** Type-Safe Enumeration

## Definition
```java
// Signature
public enum SwitchTo {
```

## Architecture & Concepts
The SwitchTo enumeration defines a fixed set of states used within the Hytale network protocol. It serves as a type-safe, human-readable representation of a specific command or visual state that must be synchronized between the client and server.

Its primary architectural role is to act as a serialization contract. Each enum constant (Disappear, PostColor, etc.) is mapped to a fixed integer value. This integer is what gets transmitted over the network for efficiency. The receiving end then uses the static factory method fromValue to deserialize the integer back into its corresponding, type-safe enum object.

This pattern prevents "magic number" bugs and ensures that only valid, known states can be processed by the protocol handlers. If an unknown integer value is received, the system fails fast by throwing a ProtocolException, protecting downstream logic from corrupted or malicious data.

### Lifecycle & Ownership
- **Creation:** All instances of SwitchTo are created and initialized by the Java Virtual Machine during class loading. This process is automatic and occurs once, typically at application startup.
- **Scope:** Application-wide. The enum constants are static, final, and globally accessible for the entire lifetime of the application.
- **Destruction:** The constants are reclaimed by the JVM only when the application shuts down. There is no manual memory management or destruction required.

## Internal State & Concurrency
- **State:** Deeply **immutable**. The internal integer value for each constant is declared final and assigned at creation. The state of a SwitchTo instance can never be modified after its construction by the JVM.
- **Thread Safety:** Inherently **thread-safe**. Due to their immutable and singleton nature, SwitchTo constants can be safely read, passed, and compared across any number of threads without requiring locks or other synchronization primitives.

## API Surface
The public contract is minimal, focusing exclusively on serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer value for network serialization. |
| fromValue(int value) | SwitchTo | O(1) | Deserializes an integer into a SwitchTo constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is intended to be used by protocol encoders and decoders when reading from or writing to a network buffer.

```java
// Example: Deserializing a state from a network packet
int stateId = networkBuffer.readVarInt();
SwitchTo receivedState = SwitchTo.fromValue(stateId);

// Use the type-safe enum in game logic
switch (receivedState) {
    case PostColor:
        // Apply post-processing color effect
        break;
    case Distortion:
        // Apply distortion effect
        break;
    // ...
}
```

### Anti-Patterns (Do NOT do this)
- **Using ordinal():** Never use the built-in `ordinal()` method for serialization. The protocol relies on the explicit `value` field. Reordering the enum declarations in the source file would change the ordinal, breaking network compatibility. The current implementation correctly avoids this.
- **Manual Value Checking:** Do not write custom `if/else` blocks to map integers to the enum. The `fromValue` method is the canonical and safest way to perform this conversion, as it includes centralized boundary and error handling.

## Data Pipeline
SwitchTo acts as a data model that represents a specific field within a larger network packet. It does not process data itself but is a critical component in the serialization and deserialization pipeline.

> **Flow (Client Ingress):**
> Network Byte Stream -> Protocol Packet Decoder -> `SwitchTo.fromValue(intValue)` -> **SwitchTo Instance** -> Game Logic Handler

