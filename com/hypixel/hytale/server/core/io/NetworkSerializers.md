---
description: Architectural reference for NetworkSerializers
---

# NetworkSerializers

**Package:** com.hypixel.hytale.server.core.io
**Type:** Utility

## Definition
```java
// Signature
public interface NetworkSerializers {
```

## Architecture & Concepts
The NetworkSerializers interface serves as a static, centralized registry for common serialization functions. Its primary architectural role is to decouple the server's internal, engine-level data structures (e.g., the mathematical Box) from the wire-format data transfer objects (DTOs) defined in the network protocol (e.g., Hitbox).

This pattern provides a canonical, single source of truth for data translation. By consolidating these conversion routines into a shared, static container, the system avoids code duplication and ensures that all parts of the server serialize core data types in a consistent manner. It functions as a specialized "Type Adapter" library, exclusively for marshalling engine state into network packets.

## Lifecycle & Ownership
- **Creation:** This interface is never instantiated. Its static fields, such as BOX, are initialized by the Java Class Loader when the NetworkSerializers class is first accessed by any part of the application.
- **Scope:** The provided serializer instances are static and exist for the entire lifetime of the server process.
- **Destruction:** The objects are eligible for garbage collection only when the application's class loader is unloaded, which typically occurs at server shutdown.

## Internal State & Concurrency
- **State:** The NetworkSerializers interface is stateless. The provided serializers are implemented as pure functions (via lambdas) that do not maintain any internal state between invocations. Their output is determined exclusively by their input.
- **Thread Safety:** All provided serializers are inherently thread-safe. As stateless, pure functions, they can be safely invoked concurrently from multiple network or game threads without requiring any external locking or synchronization mechanisms.

## API Surface
The public contract of this interface consists of its static final fields, which are pre-configured serializer instances.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BOX | NetworkSerializer<Box, Hitbox> | O(1) | A static serializer that converts an engine-side Box object into a network-protocol Hitbox packet. |

## Integration Patterns

### Standard Usage
The intended use is to pass a serializer from this class to a higher-level networking component that is responsible for dispatching packets. The networking layer uses the provided function to perform the data conversion just before writing to the network buffer.

```java
// Example: A generic network manager sending a world entity's hitbox
Box entityHitbox = entity.getCollisionBounds();
NetworkChannel channel = player.getNetworkChannel();

// The channel's send method internally uses the provided serializer
// to transform the Box into a Hitbox packet.
channel.send(entityHitbox, NetworkSerializers.BOX);
```

### Anti-Patterns (Do NOT do this)
- **Inline Re-implementation:** Do not write a custom lambda to convert a Box to a Hitbox elsewhere in the codebase. Always reference the canonical NetworkSerializers.BOX to maintain consistency. Failure to do so creates a high risk of protocol desynchronization.
- **Stateful Implementations:** Do not attempt to create custom NetworkSerializer implementations that rely on external or internal state. Serializers are expected to be pure, idempotent functions.

## Data Pipeline
NetworkSerializers acts as a critical transformation step in the outbound data pipeline, converting high-level engine concepts into low-level network structures.

> Flow:
> Engine Data (Box) -> **NetworkSerializers.BOX** -> Network DTO (Hitbox) -> Protocol Packet Encoder -> TCP/UDP Buffer

