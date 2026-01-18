---
description: Architectural reference for FXRenderMode
---

# FXRenderMode

**Package:** com.hypixel.hytale.protocol
**Type:** Value Type / Utility

## Definition
```java
// Signature
public enum FXRenderMode {
```

## Architecture & Concepts
FXRenderMode is a protocol-level enumeration that defines a strict contract for how special effects (FX) are to be rendered by the client's graphics engine. It serves as a type-safe, low-overhead mechanism for serializing specific rendering instructions from the server to the client.

This enum is fundamental to the network protocol, translating high-level rendering concepts like additive blending or screen distortion into a compact integer format suitable for network transmission. By decoupling the server's game logic from the client's specific rendering implementation, it ensures that both ends of the connection operate on a shared, unambiguous understanding of the desired visual outcome. Its primary role is to prevent data corruption and misinterpretation during the deserialization of game state packets that contain rendering commands.

## Lifecycle & Ownership
- **Creation:** The enum constants (BlendLinear, BlendAdd, etc.) are instantiated by the Java Virtual Machine during class loading. This process is automatic and occurs once when the FXRenderMode class is first referenced.
- **Scope:** As static final instances, these constants exist for the entire lifetime of the application. They are globally accessible singletons managed by the JVM.
- **Destruction:** The enum and its constants are garbage collected only when the application's ClassLoader is unloaded, which typically happens at JVM shutdown.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a single, final integer field representing its network protocol value. The state of an FXRenderMode instance can never change after its creation. The static VALUES array is also final and is used as a performance optimization for lookups.
- **Thread Safety:** Inherently thread-safe. Due to its immutable and static nature, FXRenderMode can be safely accessed and used from any thread without requiring external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the canonical integer value for serialization over the network. |
| fromValue(int value) | FXRenderMode | O(1) | Deserializes an integer from a network packet into the corresponding enum constant. Throws ProtocolException if the value is undefined. |

## Integration Patterns

### Standard Usage
This enum is almost exclusively used during the network packet deserialization process to convert a raw integer into a structured, type-safe object.

```java
// Inside a packet handler or deserializer
int renderModeId = packet.readVarInt();
try {
    FXRenderMode mode = FXRenderMode.fromValue(renderModeId);
    // Pass the 'mode' to the rendering system
    fxSystem.applyEffect(..., mode);
} catch (ProtocolException e) {
    // Handle corrupted or outdated packet data
    log.warn("Received invalid FXRenderMode ID: " + renderModeId);
}
```

### Anti-Patterns (Do NOT do this)
- **Reliance on Ordinal:** Do not use the built-in `ordinal()` method for serialization or business logic. The explicit integer assigned in the constructor is the canonical network value. The `ordinal()` value is fragile and will break the protocol if the enum constant order is ever changed.
- **Ignoring Exceptions:** The `fromValue` method is a critical validation point. Failure to catch the ProtocolException it throws can lead to client crashes or severe visual artifacts when processing malformed or legacy network packets. Always handle the exception gracefully.

## Data Pipeline
FXRenderMode acts as a deserialization and validation gateway, converting raw network data into a validated, in-memory representation for the rendering engine.

> Flow:
> Network Packet (contains int) -> Protocol Deserializer -> **FXRenderMode.fromValue()** -> Validated FXRenderMode Enum -> Graphics Rendering System

