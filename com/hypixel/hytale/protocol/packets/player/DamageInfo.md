---
description: Architectural reference for DamageInfo
---

# DamageInfo

**Package:** com.hypixel.hytale.protocol.packets.player
**Type:** Transient

## Definition
```java
// Signature
public class DamageInfo implements Packet {
```

## Architecture & Concepts
The DamageInfo class is a Data Transfer Object (DTO) that represents a single instance of damage being dealt to an entity within the game world. As part of the Hytale network protocol, its primary role is to encapsulate all necessary information for a damage event—such as the amount, cause, and source location—into a standardized binary format for transmission between the client and server.

This class is not a service or a manager; it is a pure data container. It acts as a fundamental message type within the combat and entity interaction systems. The design emphasizes performance and wire-format efficiency, utilizing a bitmask (nullBits) to compactly represent the presence of optional fields like damageSourcePosition and damageCause. This avoids sending unnecessary data over the network.

Its implementation of the Packet interface signifies its role as a top-level message that can be directly processed by the protocol's serialization and deserialization pipeline.

### Lifecycle & Ownership
- **Creation:** An instance is created under two primary scenarios:
    1. **Sending:** The game's combat logic (e.g., on the server) instantiates DamageInfo when an entity takes damage. The object is populated and passed to the network layer for serialization.
    2. **Receiving:** The network layer (e.g., a Netty pipeline handler) instantiates DamageInfo by calling the static deserialize method when an incoming network buffer with Packet ID 112 is detected.
- **Scope:** The object's lifetime is exceptionally short and bound to the scope of a single transaction. It exists only long enough to be serialized into a network buffer or to be processed by a game event handler after deserialization.
- **Destruction:** The object is eligible for garbage collection immediately after its data has been consumed. No long-term references are maintained by any core engine systems.

## Internal State & Concurrency
- **State:** The state of DamageInfo is entirely mutable. It is a simple Plain Old Java Object (POJO) whose fields can be modified after construction. It does not contain any caches or derived state.
- **Thread Safety:** This class is **not thread-safe**. It is designed for use within a single-threaded context, such as the main game loop or a dedicated network thread. Accessing or modifying a DamageInfo instance from multiple threads without external synchronization will lead to race conditions and unpredictable behavior.

## API Surface
The public contract is defined by the Packet interface and the static factory methods for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static DamageInfo | O(N) | Constructs a new DamageInfo object by reading from a ByteBuf. N is the size of the variable fields. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf for network transmission. |
| computeSize() | int | O(N) | Calculates the exact number of bytes this object will occupy when serialized. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a read-only check on a buffer to ensure it contains a valid DamageInfo structure. |
| clone() | DamageInfo | O(1) | Creates a shallow copy of the object. Note that nested objects are also cloned. |

## Integration Patterns

### Standard Usage
DamageInfo is intended to be used as an immutable message after its initial creation. The sender creates and populates it, and the receiver consumes its data.

```java
// Server-side: Creating and sending a damage packet
Vector3d explosionCenter = new Vector3d(100.0, 64.0, 100.0);
DamageCause cause = new DamageCause(DamageCause.EXPLOSION);
DamageInfo damagePacket = new DamageInfo(explosionCenter, 20.5f, cause);

// The network system would then serialize and send this packet
networkManager.sendPacket(player, damagePacket);

// Client-side: Receiving and processing the packet
// This would happen inside a network event handler
DamageInfo receivedDamage = DamageInfo.deserialize(buffer, offset);
player.getHealthComponent().applyDamage(receivedDamage.damageAmount);
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not modify a DamageInfo object after it has been passed to the network layer for sending. This can cause partial or corrupted data to be sent if the serialization happens on a different thread or at a later time.
- **Cross-Thread Sharing:** Never share a single DamageInfo instance between threads. Always create a new instance or a clone for each thread or processing context.
- **Long-Term Storage:** Do not hold references to DamageInfo objects in long-lived collections or components. They are transient data carriers and should be processed and discarded immediately.

## Data Pipeline
The DamageInfo class is a payload that moves through the network and game logic pipelines.

> **Outbound Flow (Server -> Client):**
> Combat System -> **DamageInfo (new instance)** -> Network Encoder -> `serialize()` -> ByteBuf -> TCP/UDP Socket

> **Inbound Flow (Client <- Server):**
> TCP/UDP Socket -> ByteBuf -> Protocol Dispatcher -> `deserialize()` -> **DamageInfo (rehydrated instance)** -> Game Event Bus -> Entity Health System

