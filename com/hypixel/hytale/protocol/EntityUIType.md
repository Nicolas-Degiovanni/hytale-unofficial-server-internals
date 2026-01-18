---
description: Architectural reference for EntityUIType
---

# EntityUIType

**Package:** com.hypixel.hytale.protocol
**Type:** Utility

## Definition
```java
// Signature
public enum EntityUIType {
```

## Architecture & Concepts
EntityUIType is a type-safe enumeration that defines distinct categories of user interface elements associated with in-game entities. It serves as a fundamental component of the network protocol, ensuring that both the client and server have a shared, unambiguous understanding of UI types without relying on fragile "magic numbers" or strings.

Its primary role is in the serialization and deserialization of network packets. When a packet containing entity UI data is sent, the type is encoded as a compact integer. The receiving end uses the static factory method *fromValue* to translate this integer back into a robust, type-safe EntityUIType object. This pattern is critical for protocol stability and prevents data corruption from propagating into the game state.

The inclusion of a custom ProtocolException for invalid values is a deliberate design choice for defensive programming. It allows the network layer to immediately identify and reject malformed or malicious packets, typically resulting in a connection termination.

## Lifecycle & Ownership
As a Java enumeration, EntityUIType has a lifecycle strictly managed by the JVM, distinct from standard objects.

-   **Creation:** The enum constants, EntityStat and CombatText, are instantiated automatically by the JVM when the EntityUIType class is first loaded. They are effectively static, singleton instances.
-   **Scope:** Application-level. The enum constants exist for the entire lifetime of the application and are accessible globally.
-   **Destruction:** The constants are eligible for garbage collection only when the defining class loader is unloaded, which typically occurs at application shutdown.

## Internal State & Concurrency
-   **State:** **Immutable**. Each enum constant holds a private final integer value that is assigned at creation and can never be changed. The static VALUES array is also final, providing a cached, immutable view of all constants.
-   **Thread Safety:** **Inherently thread-safe**. Due to its immutable nature, EntityUIType can be safely accessed, read, and passed between any number of threads without requiring external synchronization or locks. This guarantee is fundamental to Java enums.

## API Surface
The public contract of EntityUIType is focused on translation between the enum constant and its integer representation for network transport.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the low-level integer value corresponding to the enum constant. Used during serialization. |
| fromValue(int value) | EntityUIType | O(1) | **Critical Factory Method.** Translates a raw integer from a network stream into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
EntityUIType is almost exclusively used during the decoding of network packets to interpret a field that specifies a UI type.

```java
// In a packet decoder or handler:
// Read the raw integer representing the UI type from the network buffer.
int rawUiType = buffer.readVarInt();

// Convert the raw integer into a safe, usable enum object.
// This operation is wrapped in a try-catch block to handle protocol errors.
try {
    EntityUIType uiType = EntityUIType.fromValue(rawUiType);

    // Branch game logic based on the resolved type.
    if (uiType == EntityUIType.CombatText) {
        // Process combat text data...
    }
} catch (ProtocolException e) {
    // A protocol violation was detected. The client should be disconnected.
    connection.disconnect("Invalid EntityUIType received.");
}
```

### Anti-Patterns (Do NOT do this)
-   **Using Raw Integers:** Never use the raw integer values (0, 1) in game logic. Comparing against `uiType.getValue() == 0` defeats the purpose of type safety and makes the code brittle. Always compare against the enum constant directly: `uiType == EntityUIType.EntityStat`.
-   **Ignoring ProtocolException:** Swallowing the ProtocolException thrown by *fromValue* is a severe security and stability risk. An invalid value indicates data corruption or a protocol mismatch that must be handled immediately, typically by terminating the connection.

## Data Pipeline
EntityUIType acts as a data model for deserialization, converting raw network data into a structured, type-safe representation for use by the game engine.

> Flow:
> Network Byte Stream -> Packet Deserializer -> **EntityUIType.fromValue(intValue)** -> Game Logic -> UI Rendering System

