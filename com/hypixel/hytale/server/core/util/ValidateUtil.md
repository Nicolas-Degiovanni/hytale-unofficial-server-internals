---
description: Architectural reference for ValidateUtil
---

# ValidateUtil

**Package:** com.hypixel.hytale.server.core.util
**Type:** Utility

## Definition
```java
// Signature
public class ValidateUtil {
```

## Architecture & Concepts
ValidateUtil is a core, stateless utility class that serves as a critical data sanitization and validation layer. Its primary architectural role is to act as a gatekeeper at the boundaries of the server, particularly where data is received from untrusted sources such as network clients.

The class provides a set of pure functions to check for non-standard and potentially hazardous floating-point values, including Not-a-Number (NaN) and Infinity. These values can cause catastrophic failures, such as physics engine crashes, rendering artifacts, or infinite loops, if allowed to propagate into core game systems. By centralizing these checks, ValidateUtil enforces a consistent data integrity policy across the entire server, preventing corrupted state and improving overall system stability.

It is designed to be invoked immediately after data deserialization and before the data is passed to any business logic or game state management systems.

### Lifecycle & Ownership
As a static utility class, ValidateUtil has no instance lifecycle. It is never instantiated and therefore has no concept of ownership, scope, or destruction in the traditional object-oriented sense.

- **Creation:** The class is loaded by the JVM's ClassLoader at runtime, typically upon the first invocation of one of its static methods. No instance is ever created.
- **Scope:** Its methods are globally accessible throughout the application's lifetime once the class is loaded.
- **Destruction:** The class is unloaded only when its defining ClassLoader is garbage collected, which for core server classes, effectively means upon server shutdown.

## Internal State & Concurrency
- **State:** ValidateUtil is completely stateless. It holds no member variables and its methods operate solely on the arguments provided. Each call is an independent, deterministic operation.

- **Thread Safety:** This class is inherently thread-safe. As a stateless utility with pure functions, it can be safely called from any number of concurrent threads without requiring locks or any other synchronization primitives. Its design guarantees freedom from race conditions and memory consistency errors.

## API Surface
The public API consists of static predicates for validating numerical and composite data types.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isSafeDouble(double) | boolean | O(1) | Returns true if the value is a finite number, not NaN or Infinity. |
| isSafeFloat(float) | boolean | O(1) | Returns true if the value is a finite number, not NaN or Infinity. |
| isSafePosition(Position) | boolean | O(1) | Validates all three double components (x, y, z) of a Position object. |
| isSafeDirection(Direction) | boolean | O(1) | Validates all three float components (yaw, pitch, roll) of a Direction object. |

## Integration Patterns

### Standard Usage
ValidateUtil must be used at system boundaries to sanitize incoming data. The canonical use case is within network packet handlers immediately after a packet has been deserialized.

**WARNING:** Failure to validate incoming positional or directional data is a high-risk vulnerability that can be exploited by malicious clients to crash the server.

```java
// Example: In a network packet handler
public void handlePlayerMovePacket(PlayerMovePacket packet) {
    Position newPosition = packet.getPosition();

    // CRITICAL: Validate data before processing
    if (!ValidateUtil.isSafePosition(newPosition)) {
        // Reject the packet or disconnect the client
        log.warn("Received unsafe position data from client: " + newPosition);
        return;
    }

    // Proceed with validated data
    player.setPosition(newPosition);
}
```

### Anti-Patterns (Do NOT do this)
- **Delayed Validation:** Do not pass raw, unvalidated data from the network layer into deeper service or game logic layers. Validation must occur at the earliest possible moment. Performing validation deep within the game loop is a sign of a critical architectural flaw.
- **Redundant Checks:** Once data has been validated at the boundary, it can be considered "safe" within the server's trusted domain. Re-validating the same data in multiple internal systems is unnecessary and adds performance overhead.
- **Attempted Instantiation:** The class cannot be instantiated. Do not attempt to call `new ValidateUtil()`. All methods are static.

## Data Pipeline
ValidateUtil functions as a conditional gate in any data pipeline where external numerical inputs are processed. It either permits data to flow into the core systems or forces a rejection path.

> Flow:
> Network Packet -> Deserializer -> **ValidateUtil** -> [ (if valid) -> Game Logic -> State Update ] OR [ (if invalid) -> Rejection / Disconnect ]

