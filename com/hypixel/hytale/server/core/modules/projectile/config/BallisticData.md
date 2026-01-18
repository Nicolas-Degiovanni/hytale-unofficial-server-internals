---
description: Architectural reference for BallisticData
---

# BallisticData

**Package:** com.hypixel.hytale.server.core.modules.projectile.config
**Type:** Configuration Interface

## Definition
```java
// Signature
public interface BallisticData {
```

## Architecture & Concepts
The BallisticData interface serves as a strict data contract for defining the physical properties of a projectile. It is a core component of the server-side projectile physics module, responsible for decoupling the trajectory simulation engine from the underlying configuration sources.

By depending on this interface, the physics system can operate on any projectile type—be it an arrow, a fireball, or a custom modded entity—without knowledge of how that projectile's data is stored or loaded. This pattern ensures that the simulation logic remains pure and testable, while configuration can be sourced from various backends such as JSON files, databases, or procedurally generated values.

Implementations of this interface are expected to be simple, immutable data-transfer objects (DTOs) or value objects.

## Lifecycle & Ownership
As an interface, BallisticData itself has no lifecycle. The lifecycle described here pertains to the **concrete objects** that implement this interface.

- **Creation:** Instances are typically created by a configuration loading service during server startup or when entity assets are loaded. For example, a WeaponConfigLoader might parse a weapon's JSON definition and instantiate a concrete BallisticData object from its contents.
- **Scope:** The lifetime of an implementing object is tied to the scope of the entity configuration it represents. For standard game assets, this means the object persists for the entire server session and is shared across all instances of that entity type.
- **Destruction:** Objects are garbage collected when the server shuts down or when the associated asset definitions are unloaded (e.g., during a hot-reload or when a game module is disabled).

## Internal State & Concurrency
- **State:** The BallisticData contract implies an immutable state. Once an implementing object is constructed, its ballistic properties must not change. This is critical for predictable and deterministic physics simulations. All implementing classes must be designed as read-only.
- **Thread Safety:** Implementations must be unconditionally thread-safe. The physics engine may process entities across multiple worker threads, and it is assumed that reading ballistic properties from any thread is a safe, non-blocking operation. This is typically achieved by making the implementing class and its fields final.

## API Surface
The API surface consists exclusively of simple accessors for retrieving physical constants.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getMuzzleVelocity() | double | O(1) | Returns the initial speed of the projectile in blocks per second. |
| getGravity() | double | O(1) | Returns the gravitational acceleration applied to the projectile. |
| getVerticalCenterShot() | double | O(1) | Returns a vertical offset for the projectile's origin. |
| getDepthShot() | double | O(1) | Returns a forward offset for the projectile's origin. |
| isPitchAdjustShot() | boolean | O(1) | Determines if the projectile's initial pitch is adjusted by the engine. |

## Integration Patterns

### Standard Usage
The interface is intended to be consumed by physics systems, which receive an implementation from a higher-level manager. The consumer should never be aware of the concrete class.

```java
// Correct usage within a physics simulation service
public void calculateTrajectory(BallisticData ballisticData, Vector3 startPosition) {
    double velocity = ballisticData.getMuzzleVelocity();
    double gravity = ballisticData.getGravity();
    // ... perform physics calculations
}
```

### Anti-Patterns (Do NOT do this)
- **Mutable Implementations:** Creating a class that implements BallisticData with setters or mutable fields is a severe violation of the contract. This can lead to non-deterministic physics behavior that is extremely difficult to debug.
- **Complex Getters:** The accessor methods should not perform complex calculations, I/O, or any blocking operations. They are expected to be simple field lookups.
- **Direct Type Checking:** Code should not use `instanceof` to check for a specific implementation of BallisticData. The entire purpose of the interface is to abstract away the concrete type.

## Data Pipeline
The data defined by this interface originates in external configuration and flows into the live physics engine.

> Flow:
> Asset File (e.g., weapon.json) -> Configuration Loader/Parser -> **Concrete BallisticData Object** -> Projectile Simulation Service -> Trajectory Calculation

