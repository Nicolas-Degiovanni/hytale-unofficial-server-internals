---
description: Architectural reference for ParticleUVOption
---

# ParticleUVOption

**Package:** com.hypixel.hytale.protocol
**Type:** Static Enumeration

## Definition
```java
// Signature
public enum ParticleUVOption {
```

## Architecture & Concepts
ParticleUVOption is a type-safe enumeration that defines a fixed set of texture coordinate (UV) transformation options for the particle rendering system. Its primary architectural role is to serve as a data contract for serializing particle appearance and behavior over the network.

By mapping human-readable names like RandomFlipU to stable integer identifiers, this enum decouples the game logic from the low-level network protocol. The server can specify a particle's rendering behavior using this contract, and the client can reliably interpret and apply the correct texture transformation during rendering. This prevents "magic number" issues and ensures that both server and client operate on a shared, explicit understanding of particle effects.

The inclusion of a cached VALUES array is a performance optimization to prevent repeated memory allocation that would occur from calling the built-in `values()` method in high-frequency contexts, such as packet decoding loops.

## Lifecycle & Ownership
- **Creation:** All instances of this enum (None, RandomFlipU, etc.) are created and initialized by the Java Virtual Machine during class loading. This process is automatic and occurs only once.
- **Scope:** Application-wide. The enum constants are static, final, and exist for the entire lifetime of the application. They are effectively global constants.
- **Destruction:** The instances are garbage collected by the JVM when the application terminates and its class loader is unloaded. There is no manual destruction or cleanup required.

## Internal State & Concurrency
- **State:** **Immutable**. Each enum constant is a singleton instance with a final `value` field. Its state cannot be modified after its creation by the JVM.
- **Thread Safety:** **Inherently thread-safe**. Due to their immutable nature and the JVM's guarantees for enum initialization, all members of ParticleUVOption can be safely accessed from any thread without external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Serializes the enum constant to its stable integer representation, suitable for network transmission or storage. |
| fromValue(int value) | ParticleUVOption | O(1) | Deserializes an integer from a network packet or data store into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
The primary use case is deserializing a value from a network stream and passing it to a rendering system. The `fromValue` method is the designated factory for this purpose.

```java
// In a packet decoder, read the integer value from the buffer
int optionId = networkBuffer.readVarInt();

// Convert the integer into a type-safe enum constant
ParticleUVOption uvOption = ParticleUVOption.fromValue(optionId);

// Pass the option to the particle system for rendering logic
particleRenderer.applyUVOption(particle, uvOption);
```

### Anti-Patterns (Do NOT do this)
- **Relying on ordinal():** Do not use the built-in `ordinal()` method for serialization. The integer value returned by `ordinal()` is dependent on the declaration order of the constants in the source file. Reordering them would break network compatibility. Always use `getValue()` for a stable, explicitly defined identifier.
- **Invalid Value Handling:** Do not wrap calls to `fromValue` in a generic `try-catch (Exception e)`. The `ProtocolException` it throws is a specific, unrecoverable error indicating a corrupt or mismatched protocol version. It should be caught at a higher level to terminate the connection.

## Data Pipeline
ParticleUVOption acts as a data model within the network protocol pipeline. It does not process data itself but rather represents a state that is transmitted.

> Flow:
> Server Particle System -> Particle Definition Packet -> **ParticleUVOption (serialized as int)** -> Network Transmission -> Client Packet Decoder -> **ParticleUVOption (deserialized from int)** -> Client Particle Renderer

