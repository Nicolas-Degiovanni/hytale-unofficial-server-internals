---
description: Architectural reference for CaveBiomeMaskFlags
---

# CaveBiomeMaskFlags

**Package:** com.hypixel.hytale.server.worldgen.cave
**Type:** Utility

## Definition
```java
// Signature
public class CaveBiomeMaskFlags {
```

## Architecture & Concepts
CaveBiomeMaskFlags is a static utility class that provides a high-performance, low-level control mechanism for the server-side world generation pipeline, specifically for cave systems. It establishes a contract for controlling distinct phases of procedural generation using an integer bitmask. This approach avoids complex state objects and boolean fields in performance-critical loops.

The core architectural concept is the use of bitwise flags to represent a set of permissions for a given chunk or region within a biome. Each flag corresponds to a major stage in the cave generation process:
- **GENERATE:** The initial carving of the cave shape from the terrain.
- **POPULATE:** The placement of decorators, entities, and other features within the carved-out cave.
- **CONTINUE:** A flag indicating that cave generation can spill over into adjacent regions, allowing for large, continuous systems.

This class acts as a centralized, authoritative source for both the flag definitions and the logic required to interpret them. Higher-level systems, such as biome-specific rule engines or noise evaluators, produce an integer mask. The core cave generator then consumes this mask, using the static methods provided by this class to make decisions without needing to understand the complex rules that produced the mask.

## Lifecycle & Ownership
- **Creation:** This class is never instantiated. It is a pure utility class containing only static members. Its data is loaded into memory by the JVM ClassLoader when first referenced.
- **Scope:** Application-wide. The static members are available for the entire lifetime of the server process.
- **Destruction:** The class and its static members are unloaded when the JVM shuts down. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** Stateless and immutable. The class contains only `public static final` constants and pure, stateless static methods. It holds no mutable data and performs no caching.
- **Thread Safety:** This class is inherently thread-safe. Its stateless nature and the use of atomic integer operations (bitwise AND) ensure that all methods can be called concurrently from any thread without synchronization or risk of race conditions. It is designed for heavy use within the multi-threaded world generation system.

## API Surface
The public API consists of static constants and static test methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| GENERATE | static final int | O(1) | Flag (1) indicating permission to perform initial cave carving. |
| POPULATE | static final int | O(1) | Flag (2) indicating permission to place decorators and entities. |
| CONTINUE | static final int | O(1) | Flag (4) indicating permission for the cave system to continue into adjacent areas. |
| DEFAULT_ALLOW | static final Int2FlagsCondition | O(1) | A pre-configured condition object that always returns a mask allowing all operations (7). |
| DEFAULT_DENY | static final Int2FlagsCondition | O(1) | A pre-configured condition object that always returns a mask disallowing all operations (0). |
| canGenerate(value) | static boolean | O(1) | Returns true if the GENERATE flag is set in the provided integer mask. |
| canPopulate(value) | static boolean | O(1) | Returns true if the POPULATE flag is set in the provided integer mask. |
| canContinue(value) | static boolean | O(1) | Returns true if the CONTINUE flag is set in the provided integer mask. |
| test(value, flag) | static boolean | O(1) | The core bitwise test logic used by all other `can...` methods. |

## Integration Patterns

### Standard Usage
The primary integration pattern involves a world generation module receiving a pre-calculated integer mask from a biome rule system. The module then uses the static methods of this class to query the mask and guide its behavior.

```java
// A world generator receives a mask for a specific region.
// This mask is typically determined by higher-level biome and noise evaluation logic.
int biomeMask = CaveBiomeMaskFlags.Defaults.ALLOW_ALL; // Example: 7

// The generator checks for permission before executing expensive operations.
if (CaveBiomeMaskFlags.canGenerate(biomeMask)) {
    // Proceed with carving the main cave shape...
}

if (CaveBiomeMaskFlags.canPopulate(biomeMask)) {
    // Proceed with placing stalactites, ores, and other features...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The class has no public constructor and cannot be instantiated. Attempting `new CaveBiomeMaskFlags()` will result in a compile-time error.
- **Magic Numbers:** Do not use literal integers like `(mask & 1) == 1` in generator code. Always use the named constants (`GENERATE`, `POPULATE`) to ensure clarity and maintainability. Relying on magic numbers creates brittle code that will break if the flag values are ever changed.
- **Incorrect Bitwise Logic:** Do not attempt to check for multiple flags using a simple equality check. For example, `mask == (GENERATE | POPULATE)` is incorrect as it fails if other flags are also present. Use the provided `test` or `can...` methods.

## Data Pipeline
This class does not process a stream of data but rather interprets a single piece of data: the integer mask. It is a decision point within the larger world generation data flow.

> Flow:
> Biome Rules Engine -> Evaluates Conditions -> Produces **integer mask** -> Cave Generator -> Uses **CaveBiomeMaskFlags** to interpret mask -> Conditional Execution of Generation Stages

