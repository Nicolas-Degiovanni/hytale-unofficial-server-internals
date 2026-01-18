---
description: Architectural reference for EmptyExtraInfo
---

# EmptyExtraInfo

**Package:** com.hypixel.hytale.codec
**Type:** Singleton

## Definition
```java
// Signature
@Deprecated
public class EmptyExtraInfo extends ExtraInfo {
```

## Architecture & Concepts
The EmptyExtraInfo class is a specific implementation of the **Null Object Pattern**. It provides a non-functional, "no-op" version of its parent, ExtraInfo, which is used to track metadata during data serialization and deserialization operations within the codec system.

Its primary architectural purpose is to serve as a safe, inert placeholder when a codec operation does not require metadata tracking, validation, or path information. By providing an object that conforms to the ExtraInfo API contract but does nothing, it allows client code to avoid conditional logic and null checks.

The class is marked as **Deprecated**, indicating that its use should be limited to legacy systems or highly specific, performance-sensitive contexts where the overhead of a full metadata tracker is unacceptable. The constructor's call to its parent with `Integer.MAX_VALUE` and `ThrowingValidationResults` configures the base class for infinite recursion depth and an aggressive failure mode, though these settings are largely moot as this implementation never invokes validation.

## Lifecycle & Ownership
- **Creation:** Statically instantiated as a singleton constant, `EmptyExtraInfo.EMPTY`, upon class loading by the JVM.
- **Scope:** Application-wide. A single instance exists for the entire lifetime of the application.
- **Destruction:** The singleton instance is never explicitly destroyed. It is eligible for garbage collection only when its class loader is unloaded, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** Fundamentally **immutable and stateless**. It does not store any information passed to its methods. All calls are effectively routed to a void, and methods that return values provide a constant, empty result.
- **Thread Safety:** This class is inherently **thread-safe**. Its stateless nature ensures that the singleton instance can be shared and accessed concurrently by any number of threads without requiring locks or synchronization.

## API Surface
The public API consists entirely of no-op overrides from the ExtraInfo parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| pushKey(String key) | void | O(1) | Performs no action. This method is a no-op, effectively discarding the key. |
| popKey() | void | O(1) | Performs no action. |
| addUnknownKey(String key) | void | O(1) | Performs no action. The key is ignored. |
| getUnknownKeys() | List<String> | O(1) | Returns a statically allocated, immutable empty list. |
| peekKey() | String | O(1) | Returns the constant string literal "<empty>". |

## Integration Patterns

### Standard Usage
The sole intended use of this class is to pass its static `EMPTY` instance to codec methods that require an ExtraInfo object but for which metadata collection is not needed.

```java
// Correctly use the singleton instance for a codec operation
// where metadata is irrelevant.
SomeObject result = Codec.decode(sourceData, EmptyExtraInfo.EMPTY);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create an instance of this class using reflection. The private constructor enforces the singleton pattern. Always use the `EmptyExtraInfo.EMPTY` static final instance.
- **Expecting State:** Do not call methods like `pushKey` and expect to retrieve that information later via `peekKey` or other methods. This implementation is a black hole for data and maintains no state.
- **General Purpose Use:** Given the **@Deprecated** status, avoid using this class in new systems. Prefer a standard `CodecExtraInfo` instance unless you are working with legacy code or have a specific, profiled performance reason to avoid metadata collection.

## Data Pipeline
EmptyExtraInfo acts as a terminal node or a "sink" in the metadata pipeline. It accepts all inputs but forwards nothing, effectively terminating the flow of information.

> Flow:
> Codec Operation -> **EmptyExtraInfo** -> (Metadata Discarded)

