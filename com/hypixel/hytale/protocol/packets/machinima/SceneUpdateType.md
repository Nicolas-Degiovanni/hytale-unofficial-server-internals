---
description: Architectural reference for SceneUpdateType
---

# SceneUpdateType

**Package:** com.hypixel.hytale.protocol.packets.machinima
**Type:** Enum Constant / Utility

## Definition
```java
// Signature
public enum SceneUpdateType {
```

## Architecture & Concepts
SceneUpdateType is a type-safe enumeration that serves as a command discriminator within the machinima network protocol. Its primary function is to translate a low-level integer, representing a specific action on the wire, into a high-level, self-documenting, and immutable constant for use in game logic.

This enum is a foundational component of the machinima packet system. It ensures that both the client and server have a strict, shared understanding of the *intent* of a given scene update packet. By enforcing a constrained set of possible values (Update, Play, Stop, etc.), it eliminates the ambiguity and potential for errors associated with using "magic numbers" in the codebase. The inclusion of the `fromValue` factory method with strict bounds checking makes it a critical part of the protocol's deserialization and validation pipeline.

## Lifecycle & Ownership
-   **Creation:** Instances are created and initialized by the Java Virtual Machine (JVM) during class loading. As compile-time constants, they are not instantiated dynamically during runtime.
-   **Scope:** Application-wide. The enum constants exist for the entire lifetime of the application process.
-   **Destruction:** Instances are eligible for garbage collection only when the defining class is unloaded by the JVM, typically during application shutdown.

## Internal State & Concurrency
-   **State:** Inherently immutable. Each enum constant is a singleton instance whose internal integer `value` is final. The static `VALUES` array is initialized once during class loading and is not modified thereafter, making it effectively immutable.
-   **Thread Safety:** Fully thread-safe. As immutable singletons, SceneUpdateType constants can be safely accessed, passed, and compared across any number of threads without requiring synchronization. The static `fromValue` method is a pure function and is also completely thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the low-level integer representation of the enum constant, used for serialization to a network buffer. |
| fromValue(int value) | SceneUpdateType | O(1) | **[Deserialization Entrypoint]** Converts an integer from a network buffer into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
The primary use case is deserializing a command from a network stream and using it in a control flow structure like a switch statement to execute the correct logic.

```java
// Deserializing from a network buffer
int typeId = buffer.readVarInt();
SceneUpdateType command = SceneUpdateType.fromValue(typeId);

// Executing logic based on the command
switch (command) {
    case Play:
        machinimaSystem.playScene(sceneId);
        break;
    case Stop:
        machinimaSystem.stopScene(sceneId);
        break;
    // ... other cases
}
```

### Anti-Patterns (Do NOT do this)
-   **Integer Comparison:** Do not compare types using their raw integer values. This practice defeats the purpose of a type-safe enum, reintroduces magic numbers, and makes the code brittle.
    ```java
    // INCORRECT
    if (command.getValue() == 1) { /* ... */ }

    // CORRECT
    if (command == SceneUpdateType.Play) { /* ... */ }
    ```
-   **Unhandled Exceptions:** The `fromValue` method is a validation boundary. Failure to handle the ProtocolException it throws upon receiving an invalid integer can lead to severe state corruption, client desynchronization, or unhandled server exceptions when processing malformed or malicious packets.
    ```java
    // DANGEROUS - A malicious packet can crash this thread
    SceneUpdateType command = SceneUpdateType.fromValue(maliciousValue);

    // CORRECT - Isolate protocol failures
    try {
        SceneUpdateType command = SceneUpdateType.fromValue(valueFromPacket);
        // ... process command
    } catch (ProtocolException e) {
        log.warn("Received invalid SceneUpdateType, disconnecting client.", e);
        connection.disconnect();
    }
    ```

## Data Pipeline
SceneUpdateType acts as a deserialization and validation gate for machinima command data flowing from the network into the game engine.

> Flow:
> Network Byte Stream -> Protocol Buffer Reader -> **SceneUpdateType.fromValue(intValue)** -> Packet Object Field -> Machinima System Logic

