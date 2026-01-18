---
description: Architectural reference for SemverComparator
---

# SemverComparator

**Package:** com.hypixel.hytale.common.semver
**Type:** Transient

## Definition
```java
// Signature
public class SemverComparator implements SemverSatisfies {
```

## Architecture & Concepts
The SemverComparator is a foundational component within the Semantic Versioning (SemVer) validation system. It does not represent a version itself, but rather a *predicate* or a single, atomic version constraint. Its primary function is to encapsulate a comparison rule, such as "greater than or equal to 1.2.3".

Architecturally, this class combines a Semver object (the version to compare against) with a ComparisonType enumeration (the logical operator, e.g., >=, <, =). This design cleanly separates the data (the Semver version) from the comparison logic, adhering to the Single Responsibility Principle.

SemverComparator instances are the essential building blocks for more complex version range specifiers. A higher-level system would parse a range like `^1.2.3` or `~2.4.0` into a collection of one or more SemverComparator objects to perform a comprehensive validation.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand, typically through the static factory method `fromString`. This method parses a string representation of a constraint (e.g., ">=1.2.3") into a fully-formed object. Direct instantiation via the constructor is possible but less common, reserved for cases where the components are already parsed.
- **Scope:** The object's lifetime is ephemeral. It is designed to be created for a specific version check and is expected to be garbage collected shortly after use. It is not a long-lived service or a managed component.
- **Destruction:** Handled entirely by the Java Garbage Collector. No manual resource management is necessary.

## Internal State & Concurrency
- **State:** The SemverComparator is **Immutable**. Its internal fields, `comparisonType` and `compareTo`, are final and are assigned only once during construction. This immutability guarantees that an instance represents a consistent, unchanging rule.
- **Thread Safety:** This class is inherently **thread-safe**. Due to its immutable nature, a single instance can be safely shared and used across multiple threads without any risk of data corruption or race conditions. No external synchronization is required.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| satisfies(Semver semver) | boolean | O(1) | Executes the encapsulated comparison against the provided Semver. This is the primary logical function of the class. |
| fromString(String str) | SemverComparator | O(1) | **Static Factory.** Parses a string into a SemverComparator instance. Throws IllegalArgumentException for malformed input or NullPointerException for null input. |
| toString() | String | O(1) | Returns the canonical string representation of the comparator (e.g., ">=1.2.3"). |

## Integration Patterns

### Standard Usage
The intended use is to parse a string constraint and use the resulting object to validate one or more Semver instances.

```java
// How a developer should normally use this
Semver versionToCheck = Semver.fromString("2.0.0");
SemverComparator constraint = SemverComparator.fromString(">=1.5.0");

boolean isCompatible = constraint.satisfies(versionToCheck);
// isCompatible will be true
```

### Anti-Patterns (Do NOT do this)
- **Error Prone Construction:** Avoid manually constructing the object if you are starting from a string. The `fromString` factory contains essential parsing and validation logic.
- **Ignoring Exceptions:** The `fromString` method will throw exceptions for invalid input. Do not call it without proper error handling, as this can lead to unhandled runtime exceptions.
- **Re-parsing in Loops:** Do not call `fromString` repeatedly with the same string inside a loop. For performance, parse the constraint once and reuse the resulting immutable SemverComparator object.

## Data Pipeline
The SemverComparator acts as a processing and validation node in a data flow. It transforms a string-based rule into an executable predicate.

> Flow:
> String Constraint (e.g., ">=1.2.3") -> `SemverComparator.fromString()` -> **SemverComparator Instance** -> `satisfies(Semver)` -> Boolean Result

