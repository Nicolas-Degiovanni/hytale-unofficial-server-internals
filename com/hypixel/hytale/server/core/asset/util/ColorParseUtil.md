---
description: Architectural reference for ColorParseUtil
---

# ColorParseUtil

**Package:** com.hypixel.hytale.server.core.asset.util
**Type:** Utility

## Definition
```java
// Signature
public class ColorParseUtil {
```

## Architecture & Concepts
ColorParseUtil is a foundational, stateless utility class that serves as a dedicated codec for color data representations. Its primary architectural role is to decouple the engine's internal data structures (such as Color and ColorAlpha) from the human-readable string formats used in external asset files, particularly JSON.

This class acts as a critical bridge during the asset deserialization process. It allows artists and designers to specify colors using standard web formats (HEX, RGB, RGBA) in configuration files, while enabling the engine to work with optimized, byte-aligned color objects in performance-sensitive systems like the rendering pipeline or game logic.

The utility provides two distinct parsing pathways:
1.  **String-based Parsing:** Methods like parseColor are designed for general-purpose use where a complete color string is already available.
2.  **Stream-based Parsing:** Methods like readColor are tightly integrated with the RawJsonReader, allowing for efficient, zero-allocation parsing directly from a data stream during large file deserialization. This avoids the overhead of creating intermediate string objects.

## Lifecycle & Ownership
- **Creation:** This class is never instantiated. As a pure utility with only static members, its methods are invoked directly on the class itself. Any attempt to instantiate it via `new ColorParseUtil()` is an error.
- **Scope:** The class and its methods are available for the entire application lifetime once loaded by the Java Virtual Machine.
- **Destruction:** The class is unloaded only when the JVM shuts down. There is no instance-level state to manage or clean up.

## Internal State & Concurrency
- **State:** ColorParseUtil is **stateless** and **immutable**. It contains no instance or static fields that store data between invocations. All methods are pure functions, where the output depends solely on the input arguments.
- **Thread Safety:** This class is inherently **thread-safe**. Its stateless nature guarantees that concurrent calls from multiple threads will not interfere with each other. The pre-compiled regular expression patterns are also thread-safe for matching operations. It is safe to use this utility in multi-threaded asset loading systems without external synchronization.

## API Surface
The primary API surface consists of paired methods for parsing strings and serializing Color objects back to strings.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| parseColor(String) | Color | O(1) | Parses a standard color string (HEX, RGB) into a Color object. Returns null on failure. |
| parseColorAlpha(String) | ColorAlpha | O(1) | Parses a color string with an alpha component (HEXA, RGBA) into a ColorAlpha object. Returns null on failure. |
| readColor(RawJsonReader) | Color | O(1) | Reads and parses a color string directly from a JSON stream. Throws IOException on stream errors. |
| readColorAlpha(RawJsonReader) | ColorAlpha | O(1) | Reads and parses a color string with alpha directly from a JSON stream. Throws IOException on stream errors. |
| colorToHexString(Color) | String | O(1) | Converts a Color object into its canonical 6-digit HEX string representation (e.g., #RRGGBB). |
| colorToHexAlphaString(ColorAlpha) | String | O(1) | Converts a ColorAlpha object into its canonical 8-digit HEX string representation (e.g., #RRGGBBAA). |

## Integration Patterns

### Standard Usage
This utility should be invoked whenever color data needs to be translated from an external source into an engine-native type. The most common use case is during the deserialization of asset files.

```java
// Example: Parsing a color from a configuration string
String userDefinedColor = "#FF33AA";
Color modelColor = ColorParseUtil.parseColor(userDefinedColor);

if (modelColor == null) {
    // Handle invalid color format
    throw new ConfigurationException("Invalid color format provided.");
}

// Example: Reading from a JSON stream (conceptual)
// RawJsonReader reader = getReaderFor("my_asset.json");
// reader.findField("primaryColor");
// Color primaryColor = ColorParseUtil.readColor(reader);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not create an instance of this class. It provides no value and is a misuse of the utility pattern.
  ```java
  // ANTI-PATTERN
  ColorParseUtil util = new ColorParseUtil(); // Pointless and incorrect
  Color c = util.parseColor("#FFFFFF");
  ```
- **Reparsing in Loops:** Avoid parsing the same string repeatedly inside a performance-critical loop. Parse it once and cache the resulting Color object.
  ```java
  // ANTI-PATTERN
  for (int i = 0; i < 1000; i++) {
      // Wastes CPU cycles parsing the same string over and over
      renderObject.setColor(ColorParseUtil.parseColor("#FF0000"));
  }

  // CORRECT
  Color red = ColorParseUtil.parseColor("#FF0000");
  for (int i = 0; i < 1000; i++) {
      renderObject.setColor(red);
  }
  ```

## Data Pipeline
ColorParseUtil functions as a transformation step in the data pipeline that flows from disk to the game engine's memory.

> Flow:
> JSON Asset on Disk -> File I/O Stream -> RawJsonReader -> **ColorParseUtil** -> In-Memory Color/ColorAlpha Object -> Game Component (e.g., Material, Light)

