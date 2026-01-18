---
description: Architectural reference for SemverRange
---

# SemverRange

**Package:** com.hypixel.hytale.common.semver
**Type:** Utility / Value Object

## Definition
```java
// Signature
public class SemverRange implements SemverSatisfies {
```

## Architecture & Concepts
The SemverRange class is a core component of the engine's versioning and dependency management system. It provides a robust mechanism for parsing and evaluating semantic versioning (Semver) range specifiers, similar to those found in package managers like NPM.

Architecturally, this class embodies the **Strategy** and **Composite** design patterns. It implements the SemverSatisfies interface, which defines a single contract, `satisfies(Semver)`, for version validation. This allows different validation rules (e.g., a simple comparison, a complex range) to be treated uniformly.

A SemverRange instance is a composite structure, holding an array of other SemverSatisfies objects. This allows for the construction of complex, nested validation logic. For example, a range like `>1.0.0 <2.0.0 || >3.0.0` is parsed into a tree of comparators. The `and` boolean field determines whether all sub-comparators must be satisfied (logical AND) or if only one must be satisfied (logical OR).

This class is fundamental for ensuring compatibility between the game client, server, and third-party mods. It allows developers to declare precise version dependencies that are parsed and enforced by the engine at runtime.

### Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created via the static factory method `fromString(String)`. This method acts as the primary entry point, parsing a string representation into a fully-formed SemverRange object tree. Direct construction using `new SemverRange()` is possible but generally reserved for internal use within the parsing logic.
- **Scope:** SemverRange is an immutable value object. Its lifetime is bound to the object that holds a reference to it, such as a mod metadata container or a network protocol definition. Instances are typically short-lived, created on-demand for a specific version check and then discarded.
- **Destruction:** Managed by the Java Garbage Collector. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** **Immutable**. Once a SemverRange object is constructed, its internal state, including the `comparators` array and the `and` flag, cannot be changed. The elements within the `comparators` array are also expected to be immutable implementations of SemverSatisfies. This immutability is a critical design feature for a value object.
- **Thread Safety:** **Fully thread-safe**. Due to its immutable nature, a SemverRange instance can be safely shared across multiple threads without any external synchronization. The `satisfies` method is a pure function, guaranteeing that its execution has no side effects.

## API Surface
The public API is minimal, focusing on parsing and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fromString(String str) | SemverRange | O(L) | **Primary Factory.** Parses a string into a SemverRange object. Throws IllegalArgumentException on malformed input and NullPointerException on null input. L is the length of the input string. |
| satisfies(Semver semver) | boolean | O(N) | **Core Validator.** Checks if the given Semver instance meets the criteria defined by this range. N is the number of comparators in the range. |
| toString() | String | O(N) | Serializes the range back into its canonical string representation. |

## Integration Patterns

### Standard Usage
The intended use is to parse a version requirement string from a configuration source and use the resulting object to validate a specific version.

```java
// How a developer should normally use this
import com.hypixel.hytale.common.semver.Semver;
import com.hypixel.hytale.common.semver.SemverRange;

// 1. Define a version requirement, e.g., from a mod.json file
String requiredVersion = "^1.5.0 || >2.0.0";

// 2. Parse the requirement into a SemverRange object
SemverRange range = SemverRange.fromString(requiredVersion);

// 3. Create a Semver object for the version to be checked
Semver currentVersion = Semver.fromString("1.7.2");

// 4. Perform the validation
if (range.satisfies(currentVersion)) {
    // Version is compatible
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Avoid using `new SemverRange(comparators, and)`. The parsing logic in `fromString` is complex and handles numerous edge cases. Bypassing it is highly error-prone and breaks the intended public contract of the class.
- **Ignoring Exceptions:** The `fromString` method will throw exceptions for invalid version range syntax. Failure to handle these exceptions can lead to application crashes during configuration loading. Always wrap calls in a try-catch block when parsing user-provided or external data.
- **String Manipulation:** Do not attempt to pre-process or manipulate the version range string before passing it to `fromString`. The parser is designed to handle whitespace and various syntactical forms directly.

## Data Pipeline
SemverRange acts as a parser and a predicate in a data validation pipeline. Its primary role is to transform a declarative string rule into an executable validation object.

> Flow:
> Configuration File (e.g., mod.json) -> String representation (`"^1.2.0"`) -> **SemverRange.fromString()** -> In-memory `SemverRange` object -> `satisfies()` check with a `Semver` object -> Boolean result (Compatibility decision)

