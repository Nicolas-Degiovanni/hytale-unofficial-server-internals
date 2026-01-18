---
description: Architectural reference for Opacity
---

# Opacity

**Package:** com.hypixel.hytale.protocol
**Type:** Static Enumeration

## Definition
```java
// Signature
public enum Opacity {
```

## Architecture & Concepts
The Opacity enum is a fundamental data contract within the Hytale protocol layer. It provides a type-safe, human-readable representation for a block's light-transmitting properties. This classification is critical for the client-side rendering engine to correctly sort and process geometry.

This enum serves two primary purposes:
1.  **Protocol Serialization:** It defines the canonical integer values (0-3) that represent opacity states over the network, preventing the use of ambiguous "magic numbers" in the data stream.
2.  **Render Pipeline Sorting:** The client's rendering engine uses this value to determine which rendering pass a block belongs to. For example, geometry with Opacity.Solid is rendered in the main opaque pass, while Opacity.Transparent objects are rendered in a separate, later pass with depth writing disabled.

The static `fromValue` method acts as a deserialization factory, converting the raw integer from the network into a safe, validated enum instance. An invalid integer will result in a ProtocolException, immediately halting the processing of a malformed packet.

### Lifecycle & Ownership
-   **Creation:** All enum instances (Solid, Transparent, etc.) are created and initialized by the JVM during class loading. They are compile-time constants. The internal `VALUES` array is also populated at this time.
-   **Scope:** Application-scoped. These instances are singletons that persist for the entire lifetime of the application.
-   **Destruction:** The instances are garbage collected only when the JVM shuts down. There is no manual memory management.

## Internal State & Concurrency
-   **State:** Deeply immutable. Each enum constant holds a final primitive `value`. The static `VALUES` array is final and its contents (the enum constants themselves) are also immutable, making the entire structure constant.
-   **Thread Safety:** This enum is inherently thread-safe. Its immutability and the JVM's guarantees for static initialization ensure that it can be safely accessed and read from any thread without requiring locks or other synchronization primitives.

## API Surface
The public API is minimal, focusing on serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer representation for network serialization. |
| fromValue(int value) | Opacity | O(1) | Deserializes an integer into an Opacity instance. Throws ProtocolException if the value is out of bounds. |
| VALUES | Opacity[] | O(1) | A static, cached array of all Opacity constants. Prefer this over the `values()` method in performance-critical loops. |

## Integration Patterns

### Standard Usage
The primary use case is deserializing a block's property from a network stream and then using it in conditional logic within the rendering or game logic systems.

```java
// Example: Deserializing from a network buffer
int opacityValue = buffer.readVarInt();
Opacity blockOpacity = Opacity.fromValue(opacityValue);

// Example: Usage in the render engine
if (blockOpacity == Opacity.Transparent) {
    // Add to transparent render pass
} else {
    // Add to opaque render pass
}
```

### Anti-Patterns (Do NOT do this)
-   **Using Raw Integers:** Avoid comparing against the raw integer value. This defeats the purpose of type safety and makes the code harder to read and maintain.
    ```java
    // BAD: Relies on magic numbers
    if (block.getOpacity().getValue() == 3) { ... }

    // GOOD: Type-safe and readable
    if (block.getOpacity() == Opacity.Transparent) { ... }
    ```
-   **Manual Deserialization:** Do not implement custom logic to convert an integer to an Opacity. The built-in `fromValue` method is optimized and includes critical bounds-checking.

## Data Pipeline
Opacity is a data model, not an active processing component. It represents a state at a specific point in the data flow, primarily during the deserialization of world data.

> **Flow (Client-Side Deserialization):**
> Network Byte Stream -> Protocol Deserializer -> `int` -> **Opacity.fromValue(int)** -> Opacity Instance -> Block State -> Render Pipeline

