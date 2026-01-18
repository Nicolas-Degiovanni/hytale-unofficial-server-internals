---
description: Architectural reference for SoftParticle
---

# SoftParticle

**Package:** com.hypixel.hytale.protocol
**Type:** Type-Safe Enum

## Definition
```java
// Signature
public enum SoftParticle {
```

## Architecture & Concepts
The SoftParticle enum provides a type-safe representation for a client-side graphics setting. Its primary architectural role is to serve as a contract within the network protocol, ensuring that both the client and server agree on a fixed set of possible values for this specific configuration.

It encapsulates a simple integer value, which is its serialized form for network transmission. The class acts as a data integrity gatekeeper during deserialization. By using the static factory method fromValue, the protocol layer guarantees that any integer received from a network packet is validated against the known set of constants. This prevents invalid or malicious data from propagating into the graphics rendering engine.

The static VALUES array is a common performance optimization, caching the result of the expensive values() method call to avoid repeated array allocation during frequent deserialization operations.

## Lifecycle & Ownership
- **Creation:** Enum constants are instantiated automatically by the Java Virtual Machine (JVM) during class loading. This process is guaranteed to happen only once, before any code in the class is executed.
- **Scope:** Application-wide. The instances Enable, Disable, and Require are static, final, and globally accessible for the entire lifetime of the application.
- **Destruction:** The enum and its constants are unloaded from memory by the JVM when the application terminates. There is no manual destruction or garbage collection of enum instances.

## Internal State & Concurrency
- **State:** **Immutable**. The internal integer value of each enum constant is declared final and assigned at creation time. The state of a SoftParticle instance can never be changed after its construction by the JVM.
- **Thread Safety:** **Inherently thread-safe**. Due to their immutable nature and the JVM's guarantees about enum initialization, SoftParticle constants can be safely read and passed between any number of threads without requiring external synchronization or locks.

## API Surface
The public API is minimal, focusing exclusively on serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the primitive integer representation of the enum constant, intended for serialization into a network packet. |
| fromValue(int value) | SoftParticle | O(1) | **Critical Deserialization Method.** Converts a raw integer from a data stream into a valid SoftParticle instance. Throws ProtocolException if the integer is out of bounds. |

## Integration Patterns

### Standard Usage
The primary use case is deserializing a value received from the network and applying it to a system, such as the graphics engine.

```java
// Deserializing a graphics setting from a network buffer
int rawSetting = networkBuffer.readVarInt();
try {
    SoftParticle setting = SoftParticle.fromValue(rawSetting);
    graphicsEngine.configureSoftParticles(setting);
} catch (ProtocolException e) {
    // Handle corrupt or invalid packet data
    log.error("Received invalid SoftParticle value: " + rawSetting);
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Indexing:** Do not access the internal VALUES array directly. The fromValue method contains essential bounds-checking logic that prevents crashes and handles invalid data gracefully.
  ```java
  // BAD: Bypasses validation, can throw ArrayIndexOutOfBoundsException
  SoftParticle setting = SoftParticle.VALUES[rawValue];
  ```
- **Comparison by Value:** Do not compare enum instances by their integer value. Use direct object comparison (==) or the equals method, which is safer and more expressive.
  ```java
  // BAD: Verbose and error-prone
  if (setting.getValue() == SoftParticle.Enable.getValue()) { ... }

  // GOOD: Type-safe and guaranteed by the JVM
  if (setting == SoftParticle.Enable) { ... }
  ```

## Data Pipeline
SoftParticle acts as a data model and validation point in the network data pipeline. It does not process data itself but rather represents a validated state.

> Flow:
> Network Packet -> Protocol Buffer Decoder -> `int rawValue` -> **SoftParticle.fromValue(rawValue)** -> Graphics Engine Configuration State

