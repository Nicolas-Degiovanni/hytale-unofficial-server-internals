---
description: Architectural reference for DebugUtils
---

# DebugUtils

**Package:** com.hypixel.hytale.server.core.modules.debug
**Type:** Utility

## Definition
```java
// Signature
public class DebugUtils {
```

## Architecture & Concepts
The DebugUtils class is a server-side static utility that provides a high-level API for sending debug visualization shapes to connected clients. It serves as a crucial bridge between server-side game logic and client-side rendering for development and diagnostic purposes.

Its core architectural function is to abstract the underlying network protocol. Instead of manually constructing and broadcasting DisplayDebug or ClearDebugShapes packets, server systems can call simple, descriptive methods like addSphere or addArrow. The utility handles the creation of the appropriate packet, its serialization, and its broadcast to every player within a specified World instance.

This component is fundamentally a one-way communication channel from the server to clients. It is designed for transient, fire-and-forget visualizations and is not part of the core gameplay state replication system.

## Lifecycle & Ownership
- **Creation:** As a static utility class, DebugUtils is never instantiated. Its methods are available immediately after the class is loaded by the Java Virtual Machine.
- **Scope:** The class has an application-level scope. Its static methods can be invoked from any part of the server codebase that has access to a World object.
- **Destruction:** The class is unloaded only when the server application terminates and the JVM shuts down. There is no instance-level cleanup.

## Internal State & Concurrency
- **State:** DebugUtils is almost entirely stateless, with its methods operating exclusively on the arguments provided. However, it contains one piece of shared, mutable global state: the static boolean field DISPLAY_FORCES. This field acts as a global toggle for a specific visualization feature.

- **Thread Safety:** **WARNING:** This class is **not thread-safe**. The static field DISPLAY_FORCES can be read and written by multiple threads without any synchronization primitives. Modifying this flag from a different thread than the one calling addForce can lead to race conditions and non-deterministic behavior. Any system that modifies this flag must implement its own external synchronization. The methods themselves are re-entrant, but the class as a whole is unsafe due to this shared state.

## API Surface
The public API consists entirely of static methods for creating and managing debug shapes.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| add(world, shape, matrix, color, time, fade) | void | O(N) | The foundational method. Broadcasts a generic debug shape to all N players in the world. |
| clear(world) | void | O(N) | Broadcasts a command to clear all existing debug shapes for all N players in the world. |
| addSphere(world, pos, color, scale, time) | void | O(N) | A convenience wrapper around add for drawing a sphere at a specific position. |
| addCube(world, pos, color, scale, time) | void | O(N) | A convenience wrapper around add for drawing a cube at a specific position. |
| addArrow(world, position, direction, color, time, fade) | void | O(N) | A high-level method to visualize a vector as an arrow, composed of a cylinder and a cone. |
| addForce(world, position, force, velocityConfig) | void | O(N) | Visualizes a physics force vector. Its execution is gated by the global DISPLAY_FORCES flag. |

## Integration Patterns

### Standard Usage
DebugUtils should be called from within server-side systems, such as physics or AI updates, to provide immediate visual feedback on their internal state.

```java
// Example: Visualizing an Area of Effect (AoE) in a skill system
public void executeAoeSkill(World world, Vector3d origin) {
    double radius = 5.0;
    
    // ... skill logic ...

    // Provide visual feedback for developers
    Vector3f debugColor = new Vector3f(1.0f, 0.2f, 0.2f);
    DebugUtils.addSphere(world, origin, debugColor, radius, 5.0f);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The class has no public constructor and contains only static members. Do not attempt to create an instance with `new DebugUtils()`.
- **Unsynchronized Flag Mutation:** Do not modify the `DISPLAY_FORCES` flag from multiple threads without an external locking mechanism. This will cause unpredictable visualization behavior.
- **High-Frequency Calls:** Avoid calling these methods every tick for a large number of objects. Doing so can generate significant network traffic and potentially saturate the connection for clients, leading to lag. Use these tools for specific, targeted diagnostics.

## Data Pipeline
The data flow for this component is a simple, outbound broadcast pipeline originating from a server-side system call and terminating at the client's renderer.

> Flow:
> Server System (e.g., Physics Engine) -> **DebugUtils.addForce()** -> DisplayDebug Packet Instantiation -> Iteration over all Players in World -> PlayerRef.getPacketHandler().write() -> Network Layer Serialization -> Client-Side Deserialization -> Client-Side Rendering Engine

