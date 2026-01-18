---
description: Architectural reference for FluidFog
---

# FluidFog

**Package:** com.hypixel.hytale.protocol
**Type:** Protocol Constant

## Definition
```java
// Signature
public enum FluidFog {
   Color(0),
   ColorLight(1),
   EnvironmentTint(2);
```

## Architecture & Concepts
FluidFog is a type-safe enumeration that represents a fixed set of rendering modes for underwater or fluid-based fog effects. Its primary architectural role is to serve as a strict data contract between the client and server within the network protocol layer.

By encapsulating the fog type as an enum rather than a raw integer, the system guarantees that only valid, known fog modes can be processed by the rendering engine. The static factory method, fromValue, acts as a deserialization gatekeeper. It validates incoming integer identifiers from network packets, immediately throwing a ProtocolException upon encountering an undefined value. This fail-fast behavior is critical for protocol stability, preventing corrupted or malicious data from propagating into the game state and causing undefined rendering behavior.

This enum is a foundational element for ensuring protocol version compatibility and data integrity for rendering-related packets.

### Lifecycle & Ownership
-   **Creation:** Enum constants are instantiated by the Java Virtual Machine during the initial class loading phase. This occurs once, before any application code explicitly references the enum.
-   **Scope:** Application-wide. The instances (Color, ColorLight, EnvironmentTint) are static, final, and persist for the entire lifetime of the application.
-   **Destruction:** The enum and its constants are garbage collected only when the JVM shuts down. There is no manual memory management.

## Internal State & Concurrency
-   **State:** FluidFog is **deeply immutable**. Each enum constant holds a final integer value that is assigned at compile time and can never be changed. The static VALUES array is also final and populated only once.
-   **Thread Safety:** This class is inherently thread-safe. Its immutability guarantees that it can be read from any number of threads concurrently without the need for locks or synchronization. The static fromValue method is also reentrant and thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the raw integer value for network serialization. |
| fromValue(int value) | FluidFog | O(1) | Deserializes an integer into a FluidFog instance. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during packet serialization and deserialization, and within rendering logic to control behavior.

**Deserialization (Reading from a packet):**
```java
// In a packet handler, read the integer from the buffer
int fogTypeInt = buffer.readVarInt();

// Convert to the type-safe enum, handling potential errors
try {
    FluidFog fogType = FluidFog.fromValue(fogTypeInt);
    // Apply fogType to the game world or entity state
} catch (ProtocolException e) {
    // Disconnect client or log severe protocol error
    connection.disconnect("Invalid fluid fog type received.");
}
```

**Game Logic (Controlling rendering):**
```java
// In the rendering system, use a switch for clear, exhaustive logic
switch (currentFogType) {
    case Color:
        // Apply standard fog color
        break;
    case ColorLight:
        // Apply light-influenced fog color
        break;
    case EnvironmentTint:
        // Tint fog based on biome or environment properties
        break;
}
```

### Anti-Patterns (Do NOT do this)
-   **Using Magic Numbers:** Never use raw integers (0, 1, 2) in game logic. The primary purpose of this enum is to eliminate magic numbers and provide compile-time safety. Comparing a variable to `0` instead of `FluidFog.Color` defeats the purpose of the abstraction.
-   **Ignoring ProtocolException:** Do not wrap a call to fromValue in a generic `try-catch (Exception e)` block that silently fails. A ProtocolException indicates a severe issue, such as a client/server version mismatch or data corruption, and must be handled by terminating the connection or flagging the session as invalid.

## Data Pipeline
FluidFog acts as a translation layer between the raw network representation and the type-safe internal game state.

> **Inbound Flow (Deserialization):**
> Network Packet (Integer) -> Protocol Buffer Reader -> **FluidFog.fromValue(int)** -> Game State Object (FluidFog) -> Rendering Engine

> **Outbound Flow (Serialization):**
> Game State Object (FluidFog) -> **fluidFog.getValue()** -> Protocol Buffer Writer -> Network Packet (Integer)

