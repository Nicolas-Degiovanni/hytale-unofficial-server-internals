---
description: Architectural reference for ColorUtil
---

# ColorUtil

**Package:** com.hypixel.hytale.server.worldgen.loader.util
**Type:** Utility

## Definition
```java
// Signature
public class ColorUtil {
```

## Architecture & Concepts
ColorUtil is a stateless, static utility class designed for color space conversions. Its primary role within the engine is to provide a centralized and robust mechanism for translating human-readable hexadecimal color strings into the 24-bit integer format used by internal systems.

This component is located within the world generation loading pipeline, indicating its use in parsing configuration data such as biome definitions, block properties, or other assets where colors are specified as strings. By abstracting the parsing logic, ColorUtil decouples higher-level configuration loaders from the low-level details of integer color representation, ensuring consistency across the server.

## Lifecycle & Ownership
- **Creation:** Not applicable. As a static utility class, ColorUtil is never instantiated. The Java Virtual Machine loads the class definition into memory on first access.
- **Scope:** Application. The static methods are available globally as long as the class is loaded.
- **Destruction:** Not applicable. The class definition is unloaded by the JVM during application shutdown.

## Internal State & Concurrency
- **State:** Stateless. The class holds no internal state and all methods are pure functions, meaning their output depends solely on their inputs.
- **Thread Safety:** This class is unconditionally thread-safe. Its stateless and pure-functional nature guarantees that it can be called from any thread without risk of race conditions or the need for external synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| hexString(String s) | int | O(N) | Parses a hexadecimal string into a 24-bit RGB integer. It accepts common formats like "#RRGGBB" and "0xRRGGBB". The operation explicitly masks the result to 24 bits, stripping any potential alpha channel data. Throws NumberFormatException for invalid input. |

## Integration Patterns

### Standard Usage
This class should be used statically to convert configuration strings into usable color integers during data loading or initialization phases.

```java
// Example: Parsing a biome's sky color from a config file
String skyColorHex = biomeConfig.getString("skyColor"); // e.g., "#87CEEB"
int skyColorInt = ColorUtil.hexString(skyColorHex);

// The integer can now be used by the world generation or rendering systems.
```

### Anti-Patterns (Do NOT do this)
- **Attempted Instantiation:** The class has no public constructor and cannot be instantiated. Do not attempt to create an instance with `new ColorUtil()`. All methods must be accessed statically.
- **Invalid Input:** Passing malformed strings (e.g., "blue", "#GGHHII") will result in an unhandled `NumberFormatException`. Callers are responsible for validating input data or wrapping calls in a try-catch block if the source data cannot be trusted.

## Data Pipeline
ColorUtil acts as a transformation step within a larger data loading pipeline, typically converting raw configuration data into a format suitable for engine consumption.

> Flow:
> Configuration Asset (e.g., JSON, HOCON) -> Asset Loader -> **ColorUtil.hexString** -> In-Memory WorldGen Parameter (int)

