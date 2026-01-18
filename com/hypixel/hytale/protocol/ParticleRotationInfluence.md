---
description: Architectural reference for ParticleRotationInfluence
---

# ParticleRotationInfluence

**Package:** com.hypixel.hytale.protocol
**Type:** Constant Enum

## Definition
```java
// Signature
public enum ParticleRotationInfluence {
```

## Architecture & Concepts
The ParticleRotationInfluence enum is a core component of the Hytale network protocol, specifically for particle effect serialization. It provides a type-safe, integer-backed representation for different particle rendering behaviors related to rotation.

Its primary architectural function is to act as a data contract between the client and server. By using this enum, the game logic can operate on well-defined constants (e.g., Billboard, Velocity) while the network layer can efficiently transmit a compact integer representation. This pattern avoids the use of "magic numbers" in the application code, improving readability and reducing the risk of desynchronization errors.

The enum defines the following rendering strategies:
*   **None:** The particle has a fixed, static rotation.
*   **Billboard:** The particle's quad always faces the camera.
*   **BillboardY:** The particle's quad faces the camera, but only rotates around the Y-axis.
*   **BillboardVelocity:** The particle's quad faces the camera, but is oriented along its velocity vector.
*   **Velocity:** The particle's rotation is aligned with its direction of travel, independent of the camera.

## Lifecycle & Ownership
- **Creation:** Instances are created and managed exclusively by the Java Virtual Machine (JVM) during class loading. User code cannot and must not attempt to instantiate this enum.
- **Scope:** Application-wide static constants. The instances persist for the entire lifetime of the application.
- **Destruction:** The enum and its instances are garbage collected when the application's ClassLoader is unloaded, typically during a full shutdown.

## Internal State & Concurrency
- **State:** Fully immutable. Each enum constant is a singleton instance with a final integer field. The static VALUES array is populated once at class-load time and is not modified thereafter.
- **Thread Safety:** Inherently thread-safe. As immutable singletons managed by the JVM, instances of ParticleRotationInfluence can be safely accessed and shared across any thread without synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer value for network serialization. |
| fromValue(int value) | static ParticleRotationInfluence | O(1) | Deserializes an integer from the network into an enum constant. Throws ProtocolException for invalid values. |

## Integration Patterns

### Standard Usage
This enum is almost exclusively used during the encoding and decoding of network packets that describe particle effects.

```java
// Example: Decoding a particle effect from a network buffer
int influenceId = buffer.readVarInt();
ParticleRotationInfluence influence = ParticleRotationInfluence.fromValue(influenceId);

// ... use the influence to configure the particle renderer
```

### Anti-Patterns (Do NOT do this)
- **Using Magic Numbers:** Do not use raw integers (0, 1, 2, etc.) in game logic to represent rotation influence. This defeats the type-safety and readability benefits of the enum.
- **Ignoring Exceptions:** The fromValue method can throw a ProtocolException if it receives a malformed or out-of-bounds integer. Failure to handle this exception in a network decoding thread can lead to a channel disconnect or a thread crash. Always wrap calls to fromValue in a try-catch block during deserialization.

## Data Pipeline
ParticleRotationInfluence serves as a data model that is serialized and deserialized. It does not process data itself but represents a state within the data flow.

**Encoding Flow (Server -> Client):**
> Game State -> Particle Emitter Configuration -> **ParticleRotationInfluence.getValue()** -> Integer written to Network Buffer -> TCP Packet

**Decoding Flow (Client):**
> TCP Packet -> Network Buffer -> Integer read from Buffer -> **ParticleRotationInfluence.fromValue()** -> Particle Renderer State -> Visual Effect

