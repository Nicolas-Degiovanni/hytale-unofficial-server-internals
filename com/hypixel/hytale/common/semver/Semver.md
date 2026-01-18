---
description: Architectural reference for Semver
---

# Semver

**Package:** com.hypixel.hytale.common.semver
**Type:** Utility / Value Object

## Definition
```java
// Signature
public class Semver implements Comparable<Semver> {
```

## Architecture & Concepts
The Semver class is an immutable implementation of the Semantic Versioning 2.0.0 specification. It serves as a foundational data structure throughout the engine for version identification, comparison, and dependency resolution. Its primary role is to provide a safe, reliable, and standardized way to handle software versions, which is critical for network protocol negotiation, asset compatibility, and mod loading.

Architecturally, Semver is designed as a self-contained value object. It encapsulates not only the version data (major, minor, patch, pre-release, build) but also the complex logic for parsing from strings and comparing precedence.

A key integration point is its static `CODEC` field. This exposes the class to Hytale's serialization framework, allowing Semver instances to be effortlessly encoded and decoded for network transmission or storage on disk. This design decouples the versioning logic from the I/O layers, promoting clean separation of concerns.

## Lifecycle & Ownership
- **Creation:** Semver instances are typically created via the static factory method `fromString`. This is the preferred entry point for converting a raw string representation into a structured object. Direct instantiation using `new Semver(...)` is also possible but less common. The framework's `Codec` system will also instantiate this object during deserialization.
- **Scope:** As a value object, a Semver instance has no independent lifecycle. Its lifetime is bound to the object that holds a reference to it. They are intended to be lightweight and ephemeral.
- **Destruction:** Instances are managed by the Java Garbage Collector and are reclaimed when they are no longer referenced. No manual cleanup is required.

## Internal State & Concurrency
- **State:** The Semver class is **strictly immutable**. All internal fields are final and are set only once during construction. The `getPreRelease` method defensively returns a clone of the internal array to prevent external modification, thus preserving the integrity of the object's state.
- **Thread Safety:** Due to its immutable design, the Semver class is inherently **thread-safe**. A single instance can be safely shared, read, and compared across multiple threads without any need for external synchronization or locks.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fromString(String str) | static Semver | O(N) | Parses a string into a Semver object. Throws IllegalArgumentException on malformed input. N is the length of the string. |
| compareTo(Semver other) | int | O(M) | Compares this version against another for precedence. M is the number of pre-release identifiers. Returns -1, 0, or 1. |
| satisfies(SemverRange range) | boolean | O(1) | Checks if this version satisfies the constraints of a given SemverRange. Delegates logic to the range object. |
| toString() | String | O(M) | Serializes the object back into its canonical string representation. M is the number of version components. |

## Integration Patterns

### Standard Usage
The primary use case is parsing and comparing version strings, often during client-server handshakes or dependency checks.

```java
// How a developer should normally use this
Semver clientVersion = Semver.fromString("1.15.2-alpha.1");
Semver requiredServerVersion = Semver.fromString("1.15.0");

// Check for compatibility
if (clientVersion.compareTo(requiredServerVersion) >= 0) {
    // Proceed with connection
}
```

### Anti-Patterns (Do NOT do this)
- **String Comparison:** Never compare versions using their string representations. This ignores semantic precedence rules and will lead to incorrect behavior.
  - **BAD:** `semver1.toString().equals(semver2.toString())`
  - **GOOD:** `semver1.compareTo(semver2) == 0`
- **Manual Parsing:** Do not attempt to parse a version string by splitting it manually. The `fromString` method handles numerous edge cases, prefixes (v, =), and validation rules defined by the specification.
- **State Modification:** Do not attempt to modify a Semver instance. If a new version is needed, create a new instance.

## Data Pipeline
Semver is not a processing component in a pipeline but rather the data payload that flows through one. It is fundamental to any version-aware process, such as network authentication.

> Flow:
> Network Handshake Packet -> Codec Deserializer -> **Semver** instance -> Version Compatibility Service -> Connection Accepted/Rejected

