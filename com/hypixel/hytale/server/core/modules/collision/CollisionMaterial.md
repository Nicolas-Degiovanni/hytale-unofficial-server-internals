---
description: Architectural reference for CollisionMaterial
---

# CollisionMaterial

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Utility

## Definition
```java
// Signature
public class CollisionMaterial {
```

## Architecture & Concepts
CollisionMaterial is a foundational, static utility class that defines the physical properties of game objects for the server-side collision system. It does not contain logic; instead, it serves as a centralized, authoritative source for material type constants.

The core architectural pattern employed here is the use of **bitmasks**. Each material type is assigned a unique integer value that is a power of two. This design allows multiple properties to be combined into a single integer field using bitwise OR operations. For example, an object that is both a fluid and deals damage can be represented by the flag `MATERIAL_FLUID | MATERIAL_DAMAGE`.

This approach provides a highly efficient and memory-compact way for the physics engine to query the properties of an object during collision detection and response calculations. It decouples the physics algorithms from hardcoded "magic numbers", improving code readability and maintainability.

## Lifecycle & Ownership
- **Creation:** This class is never instantiated. As a container for static final constants, its data is loaded into memory by the Java Virtual Machine's class loader when it is first referenced by another part of the server code.
- **Scope:** The constants defined within CollisionMaterial are globally accessible and persist for the entire lifetime of the server application.
- **Destruction:** The class and its associated static data are unloaded from memory only when the JVM shuts down. No manual cleanup is required or possible.

## Internal State & Concurrency
- **State:** CollisionMaterial is stateless and immutable. All its fields are compile-time constants (`static final`), meaning their values are fixed and cannot be altered at runtime.
- **Thread Safety:** This class is inherently thread-safe. Since it contains only immutable, static data, it can be safely accessed from any number of concurrent threads without locks or other synchronization primitives.

## API Surface
The public contract of this class consists solely of its constant fields.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| MATERIAL_EMPTY | int | O(1) | Represents empty space, like air. No collision response. |
| MATERIAL_FLUID | int | O(1) | Represents liquids like water or lava that impede movement. |
| MATERIAL_SOLID | int | O(1) | Represents standard solid objects that block movement. |
| MATERIAL_SUBMERGED | int | O(1) | A state flag indicating an entity is inside a fluid volume. |
| MATERIAL_DAMAGE | int | O(1) | A property flag indicating the material inflicts damage on contact. |
| MATERIAL_SET_ANY | int | O(1) | A composite mask to check for any material type except damage. |
| MATERIAL_SET_NONE | int | O(1) | Represents the absence of any material properties. |

## Integration Patterns

### Standard Usage
The constants in CollisionMaterial are intended to be used with bitwise operators to set or check flags on a physics body.

**WARNING:** Always use bitwise AND (`&`) for checking flags, not the equality operator (`==`).

```java
// Example: Checking if a block's material flags indicate it is solid.
int blockMaterialFlags = getBlockFlags(); // e.g., returns MATERIAL_SOLID | MATERIAL_DAMAGE

// Correct way to check for a single property
if ((blockMaterialFlags & CollisionMaterial.MATERIAL_SOLID) != 0) {
    // This block is solid, proceed with collision response.
}

// Incorrect way - this will fail if other flags are present
if (blockMaterialFlags == CollisionMaterial.MATERIAL_SOLID) {
    // This code block will NOT execute if the block also has the DAMAGE flag.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never attempt to create an instance with `new CollisionMaterial()`. The class is not designed to be instantiated and has no public constructor. Access its fields statically (e.g., `CollisionMaterial.MATERIAL_SOLID`).
- **Equality Checks:** As highlighted above, using `==` to check for a material property is a common and critical error. It will lead to incorrect physics behavior if an object has multiple properties. Always use a bitwise AND check.

## Data Pipeline
CollisionMaterial is not an active component in a data pipeline; rather, it provides the definitions used to create the data. It sits at the beginning of the physics data lifecycle.

> Flow:
> Game Asset (e.g., block definition JSON) -> Server Block Loader -> **CollisionMaterial constants used to build an integer bitmask** -> Stored on In-Memory World Voxel -> Read by Physics Engine during collision query -> Collision Resolution Logic

