---
description: Architectural reference for CodecException
---

# CodecException

**Package:** com.hypixel.hytale.codec.exception
**Type:** Transient

## Definition
```java
// Signature
public class CodecException extends RuntimeException {
```

## Architecture & Concepts
The CodecException is a specialized, unchecked exception that serves as the primary failure signal for the entire data serialization and deserialization framework within the `com.hypixel.hytale.codec` package. By extending RuntimeException, it indicates catastrophic and generally unrecoverable errors in data processing, freeing consumers from mandatory `try-catch` blocks for what should be a programmatic or data integrity error.

Its core architectural purpose is to enrich standard exception reporting with high-value contextual information specific to the codec pipeline. When a data stream (e.g., BSON, JSON) or object graph fails to be processed, a CodecException is thrown, capturing not just the reason for failure but also the specific data fragment or object that caused it.

A notable design choice is the separation of the user-facing message from the detailed diagnostic message. The constructors craft a highly detailed message for the parent RuntimeException, intended for deep logging and stack trace analysis. Simultaneously, it stores a cleaner, more concise version of the primary error in its own private field, accessible via the overridden `getMessage` method. This allows for layered error handling, where different systems can access the appropriate level of detail.

## Lifecycle & Ownership
- **Creation:** A CodecException is instantiated and thrown exclusively by components within the codec subsystem. This occurs when a parser, serializer, or validator encounters a violation, such as a type mismatch, a missing required field, or malformed data syntax.
- **Scope:** The object's lifetime is ephemeral. It exists only for the duration of its propagation up the call stack from the point of the `throw` statement to the first compatible `catch` block.
- **Destruction:** The object is eligible for garbage collection immediately after the `catch` block that handles it completes execution. No part of the engine is expected to hold long-term references to a CodecException instance.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state, consisting of the `message` string, is declared `final` and set only once during construction. The state inherited from the parent `RuntimeException` is also effectively immutable post-construction.
- **Thread Safety:** **Inherently thread-safe**. As an immutable object, a CodecException instance can be safely passed across thread boundaries without any need for external synchronization. However, its creation and handling are almost always confined to the single thread responsible for the failed codec operation.

## API Surface
The primary API surface consists of its constructors, which are the designated entry points for signaling a codec failure.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CodecException(message) | Constructor | O(1) | Creates a basic exception with a simple message. |
| CodecException(message, cause) | Constructor | O(1) | Creates an exception that wraps a root cause. |
| CodecException(message, bsonValue, extraInfo, cause) | Constructor | O(1) | Enriches the exception with the specific BSON value that failed processing. |
| CodecException(message, reader, extraInfo, cause) | Constructor | O(1) | Enriches the exception with the state of the RawJsonReader at the point of failure. |
| CodecException(message, obj, extraInfo, cause) | Constructor | O(1) | Enriches the exception with the specific Java object that failed serialization. |
| getMessage() | String | O(1) | Returns the clean, concise error message, distinct from the more verbose message in the full stack trace. |

## Integration Patterns

### Standard Usage
The CodecException should be caught at a high-level boundary where data integrity failures can be gracefully handled, typically by logging the error and rejecting the invalid data packet or configuration file.

```java
// A network handler or file loader is the appropriate place to catch this
try {
    PlayerProfile profile = BsonCodecs.decode(bsonData, PlayerProfile.class);
    // ... process valid profile
} catch (CodecException e) {
    // Log the detailed error for developers and reject the data
    // The full stack trace will contain the problematic BSON fragment
    log.error("Failed to decode player profile. Data is corrupt.", e);
    connection.disconnect("Invalid profile data received.");
}
```

### Anti-Patterns (Do NOT do this)
- **Swallowing Exceptions:** Never catch a CodecException and ignore it. This indicates a severe data corruption or a mismatch between the code and the data schema, and hiding it will lead to unpredictable and unstable application state.
- **Generic Catch Blocks:** Avoid catching a broad `Exception` or `RuntimeException` when a CodecException is expected. Doing so loses the specific failure context and makes error handling less precise.
- **Using for Control Flow:** This exception signals a critical error, not a predictable, alternative path in application logic. Do not use `try-catch` blocks with CodecException to handle normal variations in data.

## Data Pipeline
The CodecException represents a terminal point in a data processing pipeline, diverting the flow to an error handling path.

> Flow:
> Invalid BSON/JSON Data -> Codec Parser -> **throw new CodecException(...)** -> Application Error Handler -> Logger / Metrics System

