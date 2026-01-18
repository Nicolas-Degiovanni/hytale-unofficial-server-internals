---
description: Architectural reference for AmbienceTransitionSpeed
---

# AmbienceTransitionSpeed

**Package:** com.hypixel.hytale.protocol
**Type:** Type-Safe Enumeration

## Definition
```java
// Signature
public enum AmbienceTransitionSpeed {
```

## Architecture & Concepts
AmbienceTransitionSpeed is a foundational component of the Hytale network protocol layer. Its primary architectural role is to provide a strongly-typed, compile-time-safe representation for a specific gameplay parameter: the rate at which environmental effects transition. This includes, but is not limited to, changes in lighting, fog density, and ambient audio.

By replacing ambiguous integer constants ("magic numbers") with explicit, self-documenting identifiers like **Default**, **Fast**, and **Instant**, this enum enhances code clarity and significantly reduces the risk of protocol-level bugs.

The class serves as a data contract for serialization and deserialization. The static factory method, fromValue, acts as the primary ingress point for converting raw integer data from a network stream or save file into a validated, in-memory object representation. This pattern ensures that any invalid data is caught at the protocol boundary, preventing propagation of corrupt state into the core game logic.

### Lifecycle & Ownership
- **Creation:** All instances (Default, Fast, Instant) are static constants, instantiated by the Java Virtual Machine during class loading. User code does not and cannot create new instances.
- **Scope:** Application-wide. The enum constants exist for the entire lifetime of the client or server process.
- **Destruction:** Instances are managed by the JVM and are reclaimed only upon process termination. There is no manual memory management or cleanup required.

## Internal State & Concurrency
- **State:** **Immutable**. Each enum constant holds a final integer value that is assigned at creation and can never be modified. The state of an AmbienceTransitionSpeed object is fixed for the lifetime of the application.
- **Thread Safety:** **Inherently thread-safe**. As immutable, globally unique constants, instances of AmbienceTransitionSpeed can be safely accessed, passed, and read by any number of threads without locks or other synchronization primitives.

## API Surface
The public contract is minimal, focusing exclusively on serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the protocol-safe integer representation of the enum constant. Used for serialization to a network stream or data file. |
| fromValue(int value) | AmbienceTransitionSpeed | O(1) | **[Factory]** Deserializes an integer into its corresponding enum constant. This is the canonical method for converting network data into a type-safe object. |

**Warning:** The fromValue method will throw a ProtocolException if the provided integer does not map to a defined constant. This is an intentional design choice to enforce protocol strictness and must be handled by the calling network code.

## Integration Patterns

### Standard Usage
This enum should be used whenever game logic needs to specify or interpret the transition speed of an ambient effect. The primary integration point is during the deserialization of world or zone data from a network packet.

```java
// Example: Deserializing a zone update packet
int speedValueFromPacket = readIntFromStream(); // Reads 1 from the network
try {
    AmbienceTransitionSpeed transitionSpeed = AmbienceTransitionSpeed.fromValue(speedValueFromPacket);

    // Pass the type-safe object to the rendering or game logic systems
    world.getActiveZone().setAmbienceTransition(newAmbience, transitionSpeed);
} catch (ProtocolException e) {
    // Handle corrupt or invalid data from the client/server
    log.error("Received invalid AmbienceTransitionSpeed: " + speedValueFromPacket);
}
```

### Anti-Patterns (Do NOT do this)
- **Using Ordinals for Serialization:** Never use the built-in `ordinal()` method for serialization. The integer value can change if the enum declaration order is modified, breaking protocol compatibility. Always use the explicit `getValue()` method.
- **Integer-Based Logic:** Avoid scattering checks for the raw integer value throughout the codebase. The purpose of the enum is to centralize this logic.

    **BAD:** `if (speed.getValue() == 1) { // logic for fast }`

    **GOOD:** `if (speed == AmbienceTransitionSpeed.Fast) { // logic for fast }`

## Data Pipeline
AmbienceTransitionSpeed acts as a data model that represents a deserialized value within a larger data flow. It ensures that raw data is validated and typed at the earliest possible stage.

> Flow:
> Network Packet (raw int) -> Protocol Deserializer -> **AmbienceTransitionSpeed.fromValue()** -> Game Logic (uses AmbienceTransitionSpeed object) -> World State -> Rendering Engine

