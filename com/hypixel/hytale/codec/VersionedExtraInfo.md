---
description: Architectural reference for VersionedExtraInfo
---

# VersionedExtraInfo

**Package:** com.hypixel.hytale.codec
**Type:** Transient

## Definition
```java
// Signature
public class VersionedExtraInfo extends ExtraInfo {
```

## Architecture & Concepts
The VersionedExtraInfo class is an implementation of the **Decorator Pattern**, designed to augment the behavior of another ExtraInfo instance. Its primary function within the codec framework is to override the version number of a delegate ExtraInfo object without altering any of its other functionality.

This component is critical for data migration and maintaining backward compatibility. When the deserialization system encounters data from a previous version, it can instantiate a standard ExtraInfo configured for the *target* data structure, then wrap it with a VersionedExtraInfo that reports the *source* data's older version number. This allows the core codec logic to correctly interpret legacy fields based on the overridden version, while the delegate ExtraInfo provides the necessary context (like the CodecStore) for the modern data format.

It acts as a lightweight, temporary context modifier during a single deserialization operation. All method calls, with the exception of getVersion, are forwarded directly to the wrapped delegate instance.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by higher-level data versioning or migration systems. It is typically created immediately before a specific codec operation begins.
- **Scope:** This object is ephemeral and its lifetime is strictly bound to the scope of the single deserialization task it was created for.
- **Destruction:** It holds no resources and is eligible for garbage collection as soon as the method using it completes and its reference goes out of scope. It does not own or manage the lifecycle of its delegate.

## Internal State & Concurrency
- **State:** The class itself is effectively immutable after construction. It holds a final integer for the version and a final reference to its delegate. However, the overall state is dependent on the mutability of the wrapped delegate ExtraInfo object.
- **Thread Safety:** This class introduces no state of its own and performs no synchronization. Its thread safety is therefore entirely determined by the thread safety of the delegate ExtraInfo instance it wraps. As codec operations are almost always confined to a single thread, this is not a practical concern, but concurrent modification of the delegate from another thread during a read operation would be catastrophic.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| VersionedExtraInfo(int, ExtraInfo) | constructor | O(1) | Constructs the decorator, capturing the version override and the delegate. |
| getVersion() | int | O(1) | Returns the explicit version number provided at construction, overriding the delegate. |
| *all other methods* | *various* | O(1) | All other public methods are direct pass-through calls to the delegate instance. |

## Integration Patterns

### Standard Usage
The canonical use case is to facilitate the reading of an older data version. A data migration utility creates a VersionedExtraInfo to wrap a standard ExtraInfo, allowing the codec to correctly process the legacy format.

```java
// Assume 'data' is a payload with an older version schema
int oldVersion = data.getVersion();
ExtraInfo modernContext = new DefaultExtraInfo(codecStore);

// Wrap the modern context with the old version number
ExtraInfo versionedContext = new VersionedExtraInfo(oldVersion, modernContext);

// The codec now uses the versioned context to read the old data
MyObject result = codec.read(data.getPayload(), versionedContext);
```

### Anti-Patterns (Do NOT do this)
- **Long-Lived Instances:** Do not store instances of VersionedExtraInfo in caches or long-lived objects. They are intended for immediate, single-use operations.
- **Stateful Delegation:** Wrapping a delegate that is being concurrently modified by another system is not supported and will lead to unpredictable deserialization errors.
- **Unnecessary Wrapping:** If the version does not need to be overridden, do not use this class. Using it with the same version as the delegate provides no value and adds a minor performance overhead.

## Data Pipeline
This class functions as a context modifier within the data deserialization pipeline. It does not process the data stream itself but provides critical metadata that guides how other components in the pipeline interpret that stream.

> Flow:
> Serialized Byte Stream -> Codec -> **VersionedExtraInfo** (Context Provider) -> Deserializer Logic -> In-Memory Game Object

