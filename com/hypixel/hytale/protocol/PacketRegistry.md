---
description: Architectural reference for PacketRegistry
---

# PacketRegistry

**Package:** com.hypixel.hytale.protocol
**Type:** Utility

## Definition
```java
// Signature
public final class PacketRegistry {
```

## Architecture & Concepts
The PacketRegistry is the definitive, static authority for all network packet definitions within the Hytale protocol. It functions as a central, application-wide implementation of the **Registry** pattern, mapping raw integer packet identifiers to their corresponding concrete Java class representations and deserialization logic.

This class acts as the critical translation layer between the low-level network pipeline and the high-level game logic. The network stack, which operates on byte streams and packet IDs, uses this registry to decode incoming data into strongly-typed Packet objects. This decouples the network transport layer from the game systems, allowing each to evolve independently. All packet types, their unique network IDs, size constraints, and validation rules are declared here, making it the single source of truth for the entire communication protocol.

**WARNING:** The contents of this registry are hardcoded and fundamental to client-server compatibility. Any modification requires a coordinated update of both client and server applications.

## Lifecycle & Ownership
- **Creation:** The PacketRegistry is not instantiated. Its internal state is populated exactly once within a static initializer block when the class is first loaded by the Java Virtual Machine. This process is guaranteed by the JLS to be atomic and thread-safe.
- **Scope:** The registry's data is global and persists for the entire lifetime of the application. It is loaded early in the application bootstrap sequence and is never unloaded or modified thereafter.
- **Destruction:** The registry and its contents are reclaimed by the JVM during application shutdown.

## Internal State & Concurrency
- **State:** The core state consists of two static final HashMaps: one for mapping integer IDs to PacketInfo records (BY_ID), and a reverse map for class types to integer IDs (BY_TYPE). This state is **mutable only during the static initialization phase**. After class loading is complete, the state is effectively immutable. A public, unmodifiable view of the primary map is exposed via the all() method to prevent external changes.
- **Thread Safety:** This class is **thread-safe**. All write operations occur within the JVM-synchronized static initializer. All public methods are read-only operations on the pre-populated maps. Concurrent reads from multiple threads are safe and do not require external synchronization.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getById(int id) | PacketInfo | O(1) | Retrieves the packet metadata for a given network ID. Returns null if the ID is not registered. |
| getId(Class type) | Integer | O(1) | Performs a reverse lookup to find the network ID for a given Packet class. Returns null if the class is not registered. |
| all() | Map | O(1) | Returns a thread-safe, unmodifiable view of the entire ID-to-PacketInfo registry. |

## Integration Patterns

### Standard Usage
The PacketRegistry is almost exclusively used by the network decoding pipeline to transform a raw ByteBuf into a specific Packet object.

```java
// Conceptual example within a network decoder
int packetId = inboundBuffer.readInt();
PacketRegistry.PacketInfo info = PacketRegistry.getById(packetId);

if (info != null) {
    // Use the registered deserializer to construct the packet
    Packet packet = info.deserialize().apply(inboundBuffer, bufferLength);
    // Pass the typed packet object to the next handler
    pipeline.fireChannelRead(packet);
}
```

### Anti-Patterns (Do NOT do this)
- **Runtime Modification:** Do not attempt to modify the registry after application startup. The design explicitly prevents this by making the registration method private and calling it only from a static block. The protocol is fixed at compile time.
- **Direct Instantiation:** Do not use `new PacketRegistry()`. The class has a private constructor and is designed for static access only.
- **Hardcoding IDs:** Avoid using magic numbers for packet IDs in game logic. Always use the type-safe reverse lookup to maintain correctness and readability.
    - **BAD:** `if (packet.getId() == 104) { ... }`
    - **GOOD:** `if (packet instanceof JoinWorld) { ... }` or `int id = PacketRegistry.getId(JoinWorld.class);`

## Data Pipeline
The PacketRegistry is a key component in the inbound network data pipeline, responsible for identifying and enabling the deserialization of packets.

> Flow:
> Raw Network ByteBuf -> Protocol Frame Decoder -> **PacketRegistry** (ID Lookup) -> Packet Deserializer -> Typed Packet Object -> Game Event Bus

