---
description: Architectural reference for CodecValidationException
---

# CodecValidationException

**Package:** com.hypixel.hytale.codec.exception
**Type:** Transient

## Definition
```java
// Signature
public class CodecValidationException extends CodecException {
```

## Architecture & Concepts
The CodecValidationException is a specialized exception type that signals a semantic failure during data deserialization. It is a critical component of the engine's data integrity and error handling strategy within the codec subsystem.

Unlike its parent, CodecException, which typically indicates a syntactic or structural error (e.g., malformed JSON), a CodecValidationException is thrown when the data is syntactically correct but fails to meet predefined business logic or schema constraints. This distinction is crucial for robust error recovery.

Common validation failures include:
*   A required field is missing.
*   A numeric value is outside its permitted range.
*   A string does not match a required pattern.
*   An enumeration value is unrecognized.

By providing a distinct exception type, the system allows high-level callers to differentiate between fundamentally corrupt data and data that is merely invalid. This enables more sophisticated error handling, such as providing default values for invalid fields versus rejecting a corrupt data packet entirely.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by low-level Codec implementations or their associated validation helpers during a deserialization process. It is never created directly by application-level services.
- **Scope:** Ephemeral. Its lifecycle is confined to the call stack from the point it is thrown to the point it is caught. It is a short-lived signal, not a persistent state object.
- **Destruction:** The object is eligible for garbage collection immediately after its corresponding catch block completes execution and it falls out of scope.

## Internal State & Concurrency
- **State:** The object's state is established at creation via its constructor and is effectively immutable. It inherits the ability to hold a detailed error message, a reference to the source data (BsonValue, RawJsonReader), and contextual metadata from its parent, CodecException. This state is intended for diagnostic and logging purposes.
- **Thread Safety:** Exception objects are thread-safe by convention. They are designed to be created, thrown, and caught within the context of a single thread. Sharing exception instances across threads is a severe anti-pattern and will lead to unpredictable behavior. The internal state is not protected by locks as concurrent access is not a supported use case.

## API Surface
The primary public contract of this class is its type, which is used in catch blocks for targeted error handling. The constructors are the interface for its creation by the codec framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CodecValidationException(String) | Constructor | O(1) | Creates an exception with a simple message. |
| CodecValidationException(String, Throwable) | Constructor | O(1) | Creates an exception that wraps a root cause. |
| CodecValidationException(String, BsonValue, ...) | Constructor | O(1) | Creates an exception with rich context from a BSON source. |
| CodecValidationException(String, RawJsonReader, ...) | Constructor | O(1) | Creates an exception with rich context from a JSON source. |
| CodecValidationException(String, Object, ...) | Constructor | O(1) | Creates an exception with context from a generic Java object. |

## Integration Patterns

### Standard Usage
The only correct way to interact with this exception is to catch it specifically to handle data validation failures separately from general parsing errors.

```java
// How a developer should handle this exception
try {
    PlayerProfile profile = codec.decode(inputStream);
    // ... process valid profile
} catch (CodecValidationException e) {
    // The data was well-formed but invalid (e.g., "level" was -5)
    // Log the specific validation error and apply corrective action.
    log.warn("Player profile validation failed, loading defaults. Reason: {}", e.getMessage());
    return PlayerProfile.createDefault();
} catch (CodecException e) {
    // The data was malformed (e.g., invalid JSON syntax)
    // This is a more severe error; the data is unreadable.
    log.error("Failed to parse player profile, data is corrupt.", e);
    throw new UnrecoverableDataException("Corrupt player profile detected", e);
}
```

### Anti-Patterns (Do NOT do this)
- **Generic Catch:** Catching `Exception` or the parent `CodecException` prevents you from distinguishing between a recoverable validation error and an unrecoverable parsing error. Always catch the most specific exception type first.
- **Application-Level Instantiation:** Never `throw new CodecValidationException()` from game logic or services. This exception is reserved for the codec layer. Application-level validation should use its own dedicated exception types.
- **Ignoring the Exception:** A silent `catch` block that does nothing is dangerous. It hides data integrity issues that can lead to `NullPointerException`, corrupted game state, or security vulnerabilities later in the program.

## Data Pipeline
This exception represents a failure path within a data deserialization pipeline. It interrupts the normal flow of data.

> Flow:
> Raw Data (JSON/BSON) -> Codec Parser -> Schema/Rule Validator -> **CodecValidationException Thrown** -> Catch Block -> Error Handling Logic (e.g., Logger, Default Value Provider)

