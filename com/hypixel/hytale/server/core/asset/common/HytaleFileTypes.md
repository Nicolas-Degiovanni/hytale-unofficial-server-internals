---
description: Architectural reference for HytaleFileTypes
---

# HytaleFileTypes

**Package:** com.hypixel.hytale.server.core.asset.common
**Type:** Utility

## Definition
```java
// Signature
public class HytaleFileTypes {
```

## Architecture & Concepts
The HytaleFileTypes class is a static constant container that serves as the single source of truth for asset file extensions within the server's asset management system. Its primary architectural purpose is to eliminate the use of "magic strings" for file types across the codebase.

By centralizing these definitions, it ensures that any component performing file I/O, asset parsing, or type validation operates on a consistent and authoritative set of identifiers. This class is a foundational, low-level utility that underpins the entire asset loading and processing pipeline, providing stability and preventing subtle bugs related to string mismatches or typos.

## Lifecycle & Ownership
- **Creation:** This class is never instantiated. As a container for static final fields, it is loaded into memory by the JVM ClassLoader the first time it is referenced by another class, such as an AssetLoader or a configuration parser.
- **Scope:** The class and its constants exist for the entire lifetime of the Java Virtual Machine. Its scope is global and static.
- **Destruction:** The class is unloaded from memory only when the JVM process terminates. No manual resource management is required.

## Internal State & Concurrency
- **State:** HytaleFileTypes is entirely stateless. It contains only static final String constants, which are immutable by definition. It holds no instance state and performs no computation.
- **Thread Safety:** This class is inherently thread-safe. Its constants can be safely read from any number of concurrent threads without requiring synchronization mechanisms.

## API Surface
The public contract of this class consists exclusively of its static fields. These fields are package-private, restricting their use to the asset management subsystem.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ANIMATION_FILE_TYPE | String | Constant | The file extension for Hytale animation files. |
| PNG_FILE_TYPE | String | Constant | The file extension for Portable Network Graphics images. |
| SVG_FILE_TYPE | String | Constant | The file extension for Scalable Vector Graphics images. |
| MODEL_FILE_TYPE | String | Constant | The file extension for Hytale 3D model files. |
| OGG_FILE_TYPE | String | Constant | The file extension for Ogg Vorbis audio files. |
| JSON_FILE_TYPE | String | Constant | The file extension for JSON data and configuration files. |

## Integration Patterns

### Standard Usage
Components should reference the static constants directly for any logic involving file extensions, such as in file filters or type dispatchers.

```java
// Correctly check if an asset path refers to a model
public boolean isModelAsset(String assetPath) {
    if (assetPath == null) {
        return false;
    }
    return assetPath.endsWith("." + HytaleFileTypes.MODEL_FILE_TYPE);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of this class. It provides no value and pollutes the heap. The class should have a private constructor to prevent this, but currently does not.
- **Hardcoding Strings:** Do not use literal strings like "png" or "json" for file type checks. This creates a maintenance liability and circumvents the purpose of this centralized utility.

**WARNING:** Failure to use the constants from HytaleFileTypes can lead to assets failing to load if the canonical file extension is ever changed in the future.

## Data Pipeline
This class does not process data itself. Instead, it provides metadata that other components in a data pipeline use for routing and validation.

> Flow:
> Asset Request -> AssetLoader -> **HytaleFileTypes** (Used for type validation) -> Specific Parser (e.g., PngParser) -> Game Engine

