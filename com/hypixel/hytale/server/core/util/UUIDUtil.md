---
description: Architectural reference for UUIDUtil
---

# UUIDUtil

**Package:** com.hypixel.hytale.server.core.util
**Type:** Utility

## Definition
```java
// Signature
public class UUIDUtil {
```

## Architecture & Concepts

UUIDUtil is a stateless, static utility class that provides core, server-wide functionality for handling Universally Unique Identifiers. Its primary role is to enforce consistent patterns for UUID generation and validation, preventing disparate implementations across the codebase.

The class establishes a canonical representation for an unassigned or "null" identifier through the **EMPTY_UUID** constant. This pattern is critical for safely serializing and comparing optional entity or object references without relying on nullable fields, thereby reducing the risk of NullPointerExceptions.

A key architectural feature is the **generateVersion3UUID** method. This method does **not** generate a standard, name-based Version 3 UUID. Instead, it generates a Version 4 (random) UUID and subsequently performs bitwise manipulation to force the version identifier to 3. This indicates its function as a compatibility shim, designed to produce identifiers that conform to the format expected by a legacy or external system, while still retaining the collision-resistance properties of a random UUID.

## Lifecycle & Ownership

As a static utility class, UUIDUtil is never instantiated and therefore has no lifecycle in the traditional object-oriented sense.

-   **Creation:** The class is loaded into the JVM by the ClassLoader upon its first reference by any other component in the server application. Its static members, such as EMPTY_UUID, are initialized at this time.
-   **Scope:** Application-wide. Its methods and constants are available globally for the entire duration of the server's runtime.
-   **Destruction:** The class is unloaded from the JVM when the server application terminates. No manual cleanup is required.

## Internal State & Concurrency

-   **State:** UUIDUtil is entirely stateless. It maintains no internal state between calls. The EMPTY_UUID field is a public static final constant and is immutable.
-   **Thread Safety:** This class is inherently thread-safe. All methods are pure functions that operate solely on their inputs without causing side effects or modifying shared state. It can be safely invoked from any concurrent thread without external synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EMPTY_UUID | static UUID | O(1) | A canonical constant representing a null or unassigned UUID (00000000-0000-0000-0000-000000000000). |
| generateVersion3UUID() | static UUID | O(1) | Generates a random UUID with its version bits forcibly set to 3. **Warning:** This is not a standard name-based UUID. |
| isEmptyOrNull(UUID) | static boolean | O(1) | Performs a null-safe check to determine if a UUID is either null or equal to the canonical EMPTY_UUID. |

## Integration Patterns

### Standard Usage

This utility should be used whenever a component needs to generate a new entity identifier or check for an unassigned one.

```java
// Generating a new, special-format ID for an entity
UUID newEntityId = UUIDUtil.generateVersion3UUID();

// Safely checking an optional reference from a network packet
UUID parentId = packet.getParentId();
if (UUIDUtil.isEmptyOrNull(parentId)) {
    // Handle the case where the parent is not assigned
}
```

### Anti-Patterns (Do NOT do this)

-   **Misinterpretation of Version 3:** Do not use generateVersion3UUID with the expectation of creating a deterministic, name-based identifier. It produces a random UUID. Using it for stable ID generation from a seed name will fail and lead to data corruption.
-   **Manual Null-Equivalent Checks:** Avoid writing custom logic to check for the empty UUID. This leads to inconsistent behavior and defeats the purpose of a centralized utility.
    ```java
    // BAD: Inconsistent and verbose
    if (uuid == null || uuid.equals(new UUID(0L, 0L))) {
        // ...
    }

    // GOOD: Consistent and clear
    if (UUIDUtil.isEmptyOrNull(uuid)) {
        // ...
    }
    ```
-   **Direct Instantiation:** The class contains only static members and cannot be instantiated. Attempting to use `new UUIDUtil()` will result in a compile-time error.

## Data Pipeline

This component does not participate in a data pipeline. It is a stateless utility that provides pure functions to be used *by* components within a pipeline, such as network decoders, entity managers, or serialization services.

