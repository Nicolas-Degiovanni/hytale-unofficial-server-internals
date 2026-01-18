---
description: Architectural reference for PatternUtil
---

# PatternUtil

**Package:** com.hypixel.hytale.common.util
**Type:** Utility

## Definition
```java
// Signature
public class PatternUtil {
```

## Architecture & Concepts
PatternUtil is a stateless, low-level utility class that provides common string manipulation functions. Its primary architectural role is to enforce a canonical data format for file paths and resource identifiers throughout the engine.

By centralizing path normalization logic, this class ensures that all subsystems—from the AssetManager to the network serialization layer—operate on a consistent, platform-agnostic path structure (using forward slashes). This is critical for preventing subtle, environment-specific bugs related to file system access on Windows versus UNIX-like operating systems. It is a foundational component, expected to be used by any system that ingests or produces file paths.

## Lifecycle & Ownership
As a static utility class, PatternUtil has no instance lifecycle.

- **Creation:** The class is loaded by the JVM ClassLoader when first referenced. No instance is ever created. A private constructor is expected to prevent instantiation.
- **Scope:** The class and its static methods are available for the entire lifetime of the application after being loaded.
- **Destruction:** The class is unloaded when the application's ClassLoader is garbage collected, typically upon JVM shutdown.

## Internal State & Concurrency
- **State:** PatternUtil is completely stateless. Its methods are pure functions, meaning their output depends solely on their input arguments, with no side effects.
- **Thread Safety:** The class is inherently thread-safe due to its stateless nature. Its methods can be invoked from any thread without synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| replaceBackslashWithForwardSlash(String) | String | O(n) | Normalizes a path string by replacing all backslash characters with forward slashes. Returns a new string. Throws NullPointerException if the input is null. |

## Integration Patterns

### Standard Usage
This utility should be invoked directly whenever a file path from an external source (e.g., user input, configuration file) is introduced into the system. It acts as a sanitization and normalization step.

```java
// Normalize a path before passing it to the AssetManager
String rawPath = "assets\\models\\character.hobj";
String normalizedPath = PatternUtil.replaceBackslashWithForwardSlash(rawPath);
assetManager.load(normalizedPath);
```

### Anti-Patterns (Do NOT do this)
- **Manual Replacement:** Do not implement path normalization manually using String.replace() elsewhere in the codebase. This creates redundant logic and introduces the risk of inconsistent behavior. Centralize all such operations through PatternUtil.
- **Ignoring Normalization:** Failure to normalize paths received from external or platform-specific APIs will lead to critical asset loading failures and inconsistent behavior between Windows and other operating systems.

## Data Pipeline
PatternUtil acts as a transformation stage within a larger data processing pipeline, most commonly for asset and resource loading.

> Flow:
> Raw File Path (e.g., from OS API) -> **PatternUtil.replaceBackslashWithForwardSlash** -> Normalized Path -> Asset Loader -> Engine

