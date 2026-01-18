---
description: Architectural reference for CameraActionType
---

# CameraActionType

**Package:** com.hypixel.hytale.protocol
**Type:** Type-Safe Enum

## Definition
```java
// Signature
public enum CameraActionType {
```

## Architecture & Concepts
The CameraActionType enum is a fundamental data contract within the Hytale network protocol layer. Its primary purpose is to provide a compile-time, type-safe representation for camera control commands sent from the server to the client. This component acts as a translation layer between the low-level integer identifiers used in network packets and the high-level, self-documenting constants used by the client's camera and rendering systems.

By encapsulating the protocol's magic numbers (0, 1, 2) into named constants (ForcePerspective, Orbit, Transition), this enum eliminates a significant class of bugs related to protocol mismatches and invalid data. The static factory method, fromValue, serves as the primary deserialization entry point, enforcing strict validation and ensuring that only defined actions can enter the game logic pipeline.

## Lifecycle & Ownership
As a Java enum, CameraActionType has a specialized lifecycle managed directly by the Java Virtual Machine, not by any application-level framework or dependency injection container.

-   **Creation:** The three instances (ForcePerspective, Orbit, Transition) are constructed automatically by the JVM when the CameraActionType class is first loaded. They are effectively static singletons.
-   **Scope:** Application-wide. Once loaded, these instances persist for the entire lifetime of the client or server process.
-   **Destruction:** The instances are garbage collected only when the JVM shuts down. There is no mechanism for manual destruction.

## Internal State & Concurrency
-   **State:** Deeply immutable. Each enum constant holds a private, final integer value that is set at creation time and can never be modified. The static VALUES array is populated once during class initialization and is not subsequently changed.
-   **Thread Safety:** Inherently thread-safe. Due to its complete immutability, CameraActionType can be safely accessed, passed, and read from any number of concurrent threads without requiring locks or any other synchronization primitives.

## API Surface
The public contract is minimal, focusing exclusively on serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | **Serialization.** Returns the integer value for this action, intended for writing to a network buffer. |
| fromValue(int value) | CameraActionType | O(1) | **Deserialization.** Converts a raw integer from a network packet into a CameraActionType instance. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during network packet processing. The server serializes an action using getValue, and the client deserializes it using fromValue, immediately followed by a switch statement to execute the corresponding logic.

```java
// Example: Client-side packet handler
int actionId = packet.readVarInt(); // Read from network buffer

try {
    CameraActionType action = CameraActionType.fromValue(actionId);
    CameraSystem cameraSystem = context.getService(CameraSystem.class);

    switch (action) {
        case ForcePerspective:
            cameraSystem.setPerspective(packet.readPerspectiveData());
            break;
        case Orbit:
            cameraSystem.startOrbit(packet.readOrbitData());
            break;
        case Transition:
            cameraSystem.beginTransition(packet.readTransitionData());
            break;
    }
} catch (ProtocolException e) {
    // Critical error: disconnect the client
    networkManager.disconnect("Invalid CameraActionType received: " + actionId);
}
```

### Anti-Patterns (Do NOT do this)
-   **Using Raw Integers:** Never use the raw integer values (0, 1, 2) directly in game logic. This defeats the purpose of type safety and creates brittle code that will break if the protocol changes. Always reference the enum constants like CameraActionType.ForcePerspective.
-   **Ignoring Exceptions:** The ProtocolException thrown by fromValue is a critical signal of a malformed packet or a protocol version mismatch. It must not be caught and ignored. Failure to handle this exception properly can lead to client desynchronization or crashes. The correct response is typically to disconnect the client.

## Data Pipeline
CameraActionType is a data model, not a processing stage. It represents data as it is serialized for network transit and deserialized for client-side consumption.

> **Serialization Flow (Server):**
> Game Event -> **CameraActionType.Orbit** -> `getValue()` -> `1` (int) -> Network Packet Buffer

> **Deserialization Flow (Client):**
> Network Packet Buffer -> `1` (int) -> `fromValue(1)` -> **CameraActionType.Orbit** -> Camera System Logic

