---
description: Architectural reference for RefreshInterval
---

# RefreshInterval

**Package:** com.hypixel.hytale.builtin.adventure.shop.barter
**Type:** Transient Data Model

## Definition
```java
// Signature
public class RefreshInterval {
```

## Architecture & Concepts
The RefreshInterval class is a specialized data model designed to represent a time duration, measured in days. Its primary role within the engine is to provide a type-safe and validated configuration value for systems that require periodic updates, specifically the adventure mode shop and barter systems.

Architecturally, this class is not a service or manager but rather a fundamental component of the engine's data-driven design philosophy. Its most critical feature is the static **CODEC** field. This links the class directly to the Hytale serialization framework, allowing instances of RefreshInterval to be seamlessly loaded from and saved to game data files (e.g., JSON definitions for a shop).

By encapsulating a simple integer, it elevates a primitive value into a self-validating configuration parameter. The attached validator ensures that any data loaded from disk is coherent (in this case, a positive number of days), preventing invalid state from propagating into the game logic.

### Lifecycle & Ownership
-   **Creation:** Instances are almost exclusively created by the Hytale **Codec** engine during the deserialization of game assets. A higher-level system, such as an AssetManager or a specific ShopDefinitionLoader, reads a data file which triggers the CODEC to instantiate this object. Direct manual instantiation is rare and generally discouraged.
-   **Scope:** The lifetime of a RefreshInterval instance is bound to its containing configuration object, such as a ShopDefinition. It is a transient object that exists only as long as its parent configuration is held in memory.
-   **Destruction:** The object is managed by the Java garbage collector. It is eligible for cleanup once its parent object is unloaded or goes out of scope. No manual destruction is required.

## Internal State & Concurrency
-   **State:** The internal state consists of a single integer, **days**. While the field is not declared final, the class is considered *effectively immutable* after construction. There are no public methods to modify the state post-instantiation.
-   **Thread Safety:** This class is not inherently thread-safe. However, its intended usage pattern as a read-only data container makes it safe for concurrent access. As long as an instance is not mutated after its initial creation (a practice for which no public API exists), it can be safely read by multiple threads without synchronization.

## API Surface
The public contract is minimal, focusing on data retrieval and serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | BuilderCodec | N/A | **Critical.** Static codec for data-driven instantiation. Defines the "Days" key and validation rules. |
| getDays() | int | O(1) | Returns the number of days for the refresh interval. |

## Integration Patterns

### Standard Usage
A developer will typically not create a RefreshInterval directly. Instead, it will be accessed as a property of a larger configuration object that has been loaded from game data files.

```java
// Assume a ShopDefinition object is loaded by the asset system
// This definition contains a RefreshInterval property deserialized via its CODEC
ShopDefinition shopDef = assetManager.load("shops/village_blacksmith.json");

// Access the interval data for use in game logic
int refreshCycle = shopDef.getRefreshPolicy().getInterval().getDays();

if (world.getDaysElapsed() % refreshCycle == 0) {
    // Trigger shop inventory refresh
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Avoid `new RefreshInterval(5)`. This hardcodes configuration that should be defined in external data files, defeating the purpose of the data-driven architecture.
-   **State Mutation:** Do not use reflection or other means to modify the internal **days** field after an object has been created. Game systems rely on this configuration being stable and read-only.
-   **Bypassing Validation:** The public constructor does not enforce the `days >= 1` validation rule. Bypassing the CODEC by creating an instance with `new RefreshInterval(0)` can lead to downstream errors, such as division by zero, in systems that consume this value.

## Data Pipeline
RefreshInterval acts as a data payload within the asset loading pipeline. It does not process data itself but is rather the product of a data transformation process.

> Flow:
> Game Data File (JSON) -> Asset Loading System -> **Hytale Codec Engine** -> Instantiated **RefreshInterval** -> Shop System Logic

