---
description: Architectural reference for ZonePatternProvider
---

# ZonePatternProvider

**Package:** com.hypixel.hytale.server.worldgen.zone
**Type:** Transient Factory

## Definition
```java
// Signature
public class ZonePatternProvider {
```

## Architecture & Concepts
The ZonePatternProvider is a foundational component in the server-side world generation pipeline. It functions as a pre-configured, immutable factory responsible for creating seed-specific ZonePatternGenerator instances.

Its core architectural purpose is to encapsulate the high-level ruleset, or *template*, for a world's zone layout. This includes the set of all possible standard zones, special unique zones, and the MaskProvider that defines placement constraints. By holding this static configuration, it cleanly separates the declarative "what" of world structure from the procedural "how" of its generation.

A single ZonePatternProvider instance, representing a specific dimension or world type, can be used to produce an infinite number of unique world layouts by supplying different seeds to its factory method. This makes it a critical link between world configuration data and the stateful, dynamic process of chunk generation.

### Lifecycle & Ownership
- **Creation:** A ZonePatternProvider is instantiated once during the server's world generation initialization phase. It is typically constructed by a higher-level configuration manager that assembles its dependencies (e.g., IPointGenerator, Zone definitions) from game data files.
- **Scope:** This object is long-lived. It persists for the entire lifetime of the world or dimension it defines. As an immutable configuration object, it is safely held as a shared resource by the world generation system.
- **Destruction:** The object is marked for garbage collection when the server unloads the corresponding world or shuts down entirely. There is no explicit destruction logic.

## Internal State & Concurrency
- **State:** Immutable. All internal fields are declared final and are assigned exclusively within the constructor. The object's state is fixed upon creation, making it a read-only data holder. The calculation of maxExtent during construction is a one-time optimization to avoid repeated computation during generation.
- **Thread Safety:** This class is inherently thread-safe. Its immutability guarantees that multiple world generation threads can call its methods, particularly createGenerator, without any risk of race conditions or data corruption. Each call to createGenerator produces a new, independent ZonePatternGenerator instance, ensuring thread isolation at the generator level.

## API Surface
The public API is minimal, focusing on its role as a factory and a provider of the underlying configuration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| createGenerator(int seed) | ZonePatternGenerator | O(N) | **Primary Factory Method.** Creates a new, seed-specific generator. Complexity is relative to N, the number of unique zones that must be placed. |
| getMaxExtent() | int | O(1) | Returns the pre-calculated maximum size of any feature within the configured zones. Used for determining buffer regions. |
| getZones() | Zone[] | O(1) | Returns the array of standard zone templates. |
| getMaskProvider() | MaskProvider | O(1) | Returns the base mask provider that defines initial placement constraints. |

## Integration Patterns

### Standard Usage
The intended use is to retrieve a provider from a central registry or configuration manager and use it to create a generator for a specific, seed-based generation task.

```java
// Obtain the provider for the current world/dimension
ZonePatternProvider provider = worldConfig.getZonePatternProvider();

// Create a generator for a specific world seed
int worldSeed = 12345;
ZonePatternGenerator generator = provider.createGenerator(worldSeed);

// Use the generator to determine the zone at a given coordinate
Zone zone = generator.getZoneAt(100, 200);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ZonePatternProvider()` in general game logic. This class has complex dependencies that should be assembled by a dedicated worldgen loading system from configuration files. Manual construction risks creating an invalid or incomplete world generation ruleset.
- **Reusing Generators:** The ZonePatternGenerator returned by createGenerator is stateful and intrinsically tied to the seed it was created with. Do not cache this generator and attempt to use it for a different seed or a different world. Always call createGenerator to get a fresh instance for each distinct, top-level generation task.

## Data Pipeline
The ZonePatternProvider acts as a factory in the data flow, transforming static configuration into a stateful, procedural generator.

> Flow:
> World Configuration Files -> WorldGen Loader -> **ZonePatternProvider** (Instance) -> `createGenerator(seed)` -> ZonePatternGenerator (Instance) -> Zone Placement Engine

