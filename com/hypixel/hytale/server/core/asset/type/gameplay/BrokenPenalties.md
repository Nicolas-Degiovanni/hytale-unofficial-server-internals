---
description: Architectural reference for BrokenPenalties
---

# BrokenPenalties

**Package:** com.hypixel.hytale.server.core.asset.type.gameplay
**Type:** Data Model / Value Object

## Definition
```java
// Signature
public class BrokenPenalties {
```

## Architecture & Concepts
The BrokenPenalties class is a passive data structure that represents a set of gameplay configuration values. Its sole purpose is to hold penalty multipliers for when an item of a certain type (Tool, Armor, Weapon) is in a "broken" state.

Architecturally, this class is a terminal node in the asset loading pipeline. It is not a service or a manager; it is a pure data container designed to be instantiated by the Hytale **Codec** system. The static CODEC field is the most critical component, defining the contract for how this object is deserialized from a source data file, such as a JSON asset.

The design deliberately uses the wrapper type Double instead of the primitive double. This allows for the representation of *optional* values. If a penalty is not specified in the source asset file, its corresponding field in the object will be null. The public getter methods accommodate this pattern by accepting a default value to be used in such cases.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale asset pipeline during server or client startup. The static `BuilderCodec` field is invoked by a higher-level asset manager to deserialize data into a new BrokenPenalties object. Manual instantiation is a design violation.
- **Scope:** The lifetime of a BrokenPenalties instance is bound to the parent asset that contains it. For example, if it is part of an item definition, it will remain in memory as long as that item definition is loaded. The static DEFAULT instance is a singleton that persists for the entire application lifetime.
- **Destruction:** Instances are eligible for garbage collection when their parent asset is unloaded from memory. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The internal state consists of three Double fields: tool, armor, and weapon. After an instance is created and populated by the Codec system, its state is effectively **immutable**. There are no public setters or methods to modify the internal values.
- **Thread Safety:** This class is **conditionally thread-safe**. Because its state is immutable post-creation, a single instance can be safely read by multiple threads without synchronization. The asset loading framework is responsible for ensuring that instances are safely published to all consumer threads after deserialization is complete.

## API Surface
The public contract is minimal, focusing on data retrieval and the static definitions required for serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getTool(double) | double | O(1) | Retrieves the tool penalty. Returns the provided default if the penalty is not defined in the source asset. |
| getArmor(double) | double | O(1) | Retrieves the armor penalty. Returns the provided default if the penalty is not defined in the source asset. |
| getWeapon(double) | double | O(1) | Retrieves the weapon penalty. Returns the provided default if the penalty is not defined in the source asset. |
| DEFAULT | BrokenPenalties | N/A | A static, immutable instance representing no defined penalties. Serves as a safe default value. |
| CODEC | BuilderCodec | N/A | The static codec responsible for the serialization and deserialization of this object. This is the integration point with the asset system. |

## Integration Patterns

### Standard Usage
This class is not meant to be requested from a service locator or dependency injection container. Instead, it is accessed as a property of a larger, data-driven asset, such as an item or equipment definition.

```java
// A game system retrieves a fully loaded asset
ItemDefinition swordDefinition = assetManager.get("hytale:iron_sword");

// The BrokenPenalties object is a component of that asset
BrokenPenalties penalties = swordDefinition.getBrokenPenalties(); // Hypothetical getter

// The system uses the values, providing a safe default
double durabilityPenalty = penalties.getWeapon(0.0);
if (durabilityPenalty > 0.0) {
    // Apply gameplay logic
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new BrokenPenalties()`. This bypasses the entire data-driven architecture of the engine. Game balance and configuration should be controlled through asset files, not hard-coded in Java.
- **Assuming Non-Null:** Do not access the internal fields directly (e.g., via reflection) and assume they are non-null. The core design relies on them being potentially null. Always use the provided getter methods to safely handle undefined penalties.

## Data Pipeline
The BrokenPenalties class is a product of the asset deserialization pipeline. Data flows from a configuration file on disk, through the engine's codec system, and results in an in-memory instance of this class.

> Flow:
> Asset File (e.g., JSON) -> AssetManager -> Codec Deserializer -> **BrokenPenalties Instance** -> Game System (e.g., CombatSystem)

