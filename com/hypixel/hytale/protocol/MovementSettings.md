---
description: Architectural reference for MovementSettings
---

# MovementSettings

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class MovementSettings {
```

## Architecture & Concepts
The MovementSettings class is a struct-like Data Transfer Object that encapsulates the complete set of physical properties governing an entity's movement. It acts as a data contract between the server and client, ensuring that physics simulations are consistent and server-authoritative.

Architecturally, this class sits at the boundary between the low-level network protocol layer and the high-level game simulation engine. Its sole purpose is to be serialized into and deserialized from a raw byte stream. The structure is defined with a fixed-size memory layout (251 bytes), which is critical for high-performance network communication, as it avoids the overhead of dynamic sizing or complex parsing.

Each instance represents a specific "movement profile" that can be applied to an entity. For example, a player might have a standard walking profile, a swimming profile, and a flying profile, each represented by a distinct MovementSettings object. The server sends these profiles to the client to configure the client-side physics controller.

## Lifecycle & Ownership
- **Creation:** Instances are created under two primary circumstances:
    1.  By the network deserialization pipeline via the static `deserialize` factory method when a corresponding packet is received from the network.
    2.  Programmatically by game logic, typically on the server, to define a new movement profile for an entity type or game mode. The copy constructor or `clone` method is often used to create variations of a base profile.
- **Scope:** Transient. The lifetime of a MovementSettings object is typically short. It is created, used to configure a physics component, and then becomes eligible for garbage collection. It is not a long-lived service or component.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no native resources or explicit cleanup methods.

## Internal State & Concurrency
- **State:** Highly mutable. All fields are public, allowing for direct, unchecked modification. This design prioritizes performance and ease of access over encapsulation, which is a common trade-off for internal DTOs. The object's state is a direct representation of the 251-byte network block.
- **Thread Safety:** **This class is not thread-safe.** Its public mutable fields make it inherently unsafe for concurrent access. If an instance is read by one thread while being modified by another, the reading thread may observe a corrupt or inconsistent state.

    **WARNING:** All access to a MovementSettings instance must be externally synchronized or confined to a single thread, such as the main game loop or a specific Netty event loop thread. Do not share instances across threads without proper locking.

## API Surface
The public API is focused on serialization, validation, and object creation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | MovementSettings | O(1) | **[Static]** Constructs a new instance by reading 251 bytes from the provided ByteBuf. |
| serialize(buf) | void | O(1) | Writes the object's state as 251 bytes into the provided ByteBuf. |
| validateStructure(buf, offset) | ValidationResult | O(1) | **[Static]** Checks if the buffer contains enough readable bytes to deserialize an instance. |
| clone() | MovementSettings | O(1) | Creates a deep copy of the instance. Useful for creating modified profiles. |
| computeSize() | int | O(1) | Returns the constant size of the serialized object, which is 251. |

## Integration Patterns

### Standard Usage
The most common use case is deserializing the object from a network buffer and applying it to an entity's physics state.

```java
// Executed within a Netty channel handler or network thread
ByteBuf packetData = ...;
ValidationResult result = MovementSettings.validateStructure(packetData, 0);

if (result.isOk()) {
    MovementSettings profile = MovementSettings.deserialize(packetData, 0);
    
    // Dispatch the new profile to the game thread to be applied
    gameLogic.scheduleTask(() -> {
        Entity targetEntity = world.getEntity(entityId);
        targetEntity.getPhysicsComponent().applyMovementProfile(profile);
    });
}
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Sharing a single instance across multiple threads without synchronization will lead to data corruption and unpredictable physics behavior.

    ```java
    // BAD - Race condition
    MovementSettings sharedProfile = getGlobalProfile();
    
    // Thread A
    new Thread(() -> sharedProfile.baseSpeed = 10.0f).start();
    
    // Thread B (Game Loop)
    // May read baseSpeed as the old or new value, or a torn value.
    float speed = sharedProfile.baseSpeed * multiplier;
    ```

- **In-Place Modification of Shared Profiles:** Modifying a profile that is actively being used by multiple entities will cause all of them to change behavior simultaneously. Clone the profile first if you need a variation.

    ```java
    // BAD - Unintended side effects
    MovementSettings defaultProfile = getPlayerDefaultProfile();
    Entity player1 = ...;
    Entity player2 = ...;
    player1.setMovement(defaultProfile);
    player2.setMovement(defaultProfile);
    
    // This will slow down player1 AND player2
    defaultProfile.baseSpeed *= 0.5f; 
    ```

## Data Pipeline
MovementSettings serves as a data payload within the network-to-game-engine pipeline. It translates a raw binary format into a structured, usable object for the physics simulation.

> Flow:
> Network ByteBuf -> Protocol Decoder -> **MovementSettings** (Deserialization) -> Entity Physics Component -> Game Simulation Tick

