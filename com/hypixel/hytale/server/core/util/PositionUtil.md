---
description: Architectural reference for PositionUtil
---

# PositionUtil

**Package:** com.hypixel.hytale.server.core.util
**Type:** Utility

## Definition
```java
// Signature
public class PositionUtil {
```

## Architecture & Concepts
PositionUtil is a stateless, static utility class that serves as a critical translation layer between the engine's internal mathematical representations and the network protocol's data transfer objects (DTOs). It acts as a boundary adapter, decoupling the core game simulation from the wire format used for client-server communication.

The engine's core logic operates on rich, behavior-heavy objects within the `com.hypixel.hytale.math.vector` package, such as Vector3d and Transform. These classes contain methods for complex geometric and physics calculations. In contrast, the network DTOs in the `com.hypixel.hytale.protocol` package, such as Position and Direction, are simple, flat data structures designed for efficient serialization and network transport.

This class exclusively provides pure functions to convert data between these two domains. This separation is a key architectural principle, allowing the network protocol to remain stable and simple while the internal engine mathematics can evolve independently.

### Lifecycle & Ownership
- **Creation:** As a static utility class, PositionUtil is never instantiated. Its bytecode is loaded by the JVM ClassLoader upon the first static method access.
- **Scope:** The class and its methods are available globally throughout the application's lifetime.
- **Destruction:** The class is unloaded by the JVM during application shutdown. It does not manage any resources that require explicit cleanup.

## Internal State & Concurrency
- **State:** PositionUtil is entirely **stateless**. It holds no member variables and its methods' outputs depend solely on their inputs.
- **Thread Safety:** This class is inherently **thread-safe**. Its lack of internal state means that concurrent calls from multiple threads pose no risk of race conditions or data corruption within the utility itself. Callers are still responsible for ensuring that the objects passed as arguments are not being mutated by other threads simultaneously.

## API Surface
The API is divided into two primary categories: conversion methods (prefixed with *to*) that create new objects, and assignment methods (*assign*) that mutate existing objects.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toTransformPacket(transform) | Transform | O(1) | Converts an engine Transform into a new network protocol Transform. |
| toPositionPacket(position) | Position | O(1) | Converts an engine Vector3d into a new network protocol Position. |
| toTransform(transform) | com.hypixel.hytale.math.vector.Transform | O(1) | Converts a network protocol Transform into a new engine Transform. Returns null if input is null. |
| toVector3d(position) | Vector3d | O(1) | Converts a network protocol Position into a new engine Vector3d. |
| assign(position, vector) | void | O(1) | **Mutates** the given Position object with values from the Vector3d. |
| assign(direction, vector) | void | O(1) | **Mutates** the given Direction object with values from the Vector3f. |
| equals(vector, position) | boolean | O(1) | Performs a value-based comparison between an engine vector and a protocol object. |

## Integration Patterns

### Standard Usage
The primary use case is marshalling data to and from the network layer. When preparing to send an entity's state, its internal engine Transform is converted into a protocol-safe format.

```java
// Example: Preparing an entity's transform for a network packet
com.hypixel.hytale.math.vector.Transform entityTransform = entity.getTransform();

// Convert the engine-native object to a network-ready DTO
Transform packetTransform = PositionUtil.toTransformPacket(entityTransform);

// Add the DTO to the outgoing network packet
someNetworkPacket.setTransform(packetTransform);
```

### Anti-Patterns (Do NOT do this)
- **Attempting Instantiation:** This class is not designed to be instantiated. Do not attempt `new PositionUtil()`; all methods are static and should be accessed directly.
- **Ignoring Mutability of `assign`:** The `assign` methods modify their first argument in-place. Using them on shared objects without proper synchronization or understanding can lead to unpredictable side effects and bugs that are difficult to trace. Prefer the `to...` methods unless performance constraints from object allocation are a measured bottleneck.

## Data Pipeline
PositionUtil is a key component in the data flow between the game world and the network stack.

**Outbound Flow (Engine to Network):**
> `math.vector.Transform` (Game World) -> **PositionUtil.toTransformPacket** -> `protocol.Transform` (DTO) -> Packet Serializer -> Network Buffer

**Inbound Flow (Network to Engine):**
> Network Buffer -> Packet Deserializer -> `protocol.Transform` (DTO) -> **PositionUtil.toTransform** -> `math.vector.Transform` (Game World)

