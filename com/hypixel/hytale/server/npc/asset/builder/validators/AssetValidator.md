---
description: Architectural reference for AssetValidator
---

# AssetValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class AssetValidator {
```

## Architecture & Concepts

The AssetValidator is an abstract base class that forms the foundation of the server-side asset reference validation system. It establishes a formal contract for verifying that string identifiers in configuration files correctly point to existing game assets. This component is a critical part of the server's data integrity and content loading pipeline, preventing crashes and runtime errors caused by misconfigured or missing assets, particularly within NPC and world generation definitions.

Architecturally, this class implements the **Strategy Pattern**. Each concrete subclass encapsulates a specific validation algorithm for a distinct asset type (e.g., models, sounds, particle effects). This design decouples the validation logic from the configuration parsing system that uses it, allowing new asset types to be supported without modifying the core parsing engine.

A key feature is its integration with the Hytale Codec system via the `updateSchema` method. During schema generation, the validator enriches the schema with metadata, flagging specific string fields as Hytale asset references. This metadata is essential for developer tooling, enabling features like in-editor validation, auto-completion, and asset browsing.

The `Config` enum provides a declarative mechanism to handle common validation variations, such as nullability or whether an empty string is permissible. This avoids the proliferation of subclasses for minor rule changes, promoting a cleaner and more maintainable implementation.

### Lifecycle & Ownership
-   **Creation:** Concrete implementations of AssetValidator are instantiated by the server's configuration loading framework or a dedicated validation registry during server bootstrap. They are typically defined once as static instances or singletons for each asset type.
-   **Scope:** An AssetValidator instance is designed to be long-lived. It persists for the entire server session and is reused across all configuration loading and validation tasks for its designated asset type.
-   **Destruction:** Instances are subject to standard garbage collection and are typically destroyed only upon server shutdown when the application's class loaders are released.

## Internal State & Concurrency
-   **State:** The internal state is limited to the `config` EnumSet, which is provided at construction time and is **immutable** thereafter. The class is intentionally designed to be stateless with respect to the data it validates.
-   **Thread Safety:** This class and its intended usage pattern are **thread-safe**. Because its internal state is immutable and its validation methods operate solely on their input parameters, a single instance can be safely invoked by multiple threads concurrently. This is crucial for parallelized asset loading and server initialization.

## API Surface

The public contract is defined by its abstract methods, which must be implemented by all concrete subclasses.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDomain() | String | O(1) | Returns a high-level category for the asset, e.g., "Model" or "Sound". |
| getAssetName() | String | O(1) | Returns the specific asset type identifier used for schema linking, e.g., "hytale:model". |
| test(String assetPath) | boolean | O(N) | The core validation method. Returns true if the asset path is valid. Complexity depends on the asset lookup mechanism. |
| errorMessage(String path, String context) | String | O(1) | Generates a formatted, human-readable error message for a failed validation. |
| updateSchema(StringSchema schema) | void | O(1) | Modifies a Hytale Codec schema to add asset reference metadata. |

## Integration Patterns

### Standard Usage

The primary pattern is to extend AssetValidator to create a specialized validator for a specific asset type. This subclass is then registered with and invoked by a higher-level configuration processing system.

```java
// 1. Define a concrete validator for a specific asset type, like a particle effect.
public final class ParticleValidator extends AssetValidator {

    public ParticleValidator(EnumSet<Config> config) {
        super(config);
    }

    @Override
    public String getDomain() {
        return "Particle Effect";
    }

    @Override
    public String getAssetName() {
        return "hytale:particle";
    }

    @Override
    public boolean test(String assetPath) {
        // Assumes the existence of a central, thread-safe AssetRegistry
        return AssetRegistry.INSTANCE.particleExists(assetPath);
    }

    @Override
    public String errorMessage(String assetPath, String configPath) {
        return String.format("Particle effect '%s' referenced in '%s' does not exist.", assetPath, configPath);
    }
}

// 2. Use the validator within a configuration loading service.
// This logic would typically reside deep within the server's asset builder framework.
AssetValidator particleValidator = new ParticleValidator(EnumSet.noneOf(Config.class));
String particleRefFromConfig = "particles/magic/fireball.json";

if (!particleValidator.test(particleRefFromConfig)) {
    String error = particleValidator.errorMessage(particleRefFromConfig, "npc/mage.json");
    throw new ConfigurationException(error);
}
```

### Anti-Patterns (Do NOT do this)
-   **Mutable State in Subclasses:** Introducing mutable fields into a subclass is a severe anti-pattern. It breaks the stateless design contract and immediately invalidates thread safety, which can lead to unpredictable race conditions during parallel server loading.
-   **Implementing Null/Empty Checks in `test`:** The `test` method should *not* contain logic for checking if a value is null or empty. This responsibility is handled by the configuration framework using the `isNullable()` and `canBeEmpty()` flags. Duplicating this logic leads to inconsistent behavior.
-   **Expensive Operations in Constructors:** Constructors should be lightweight and do nothing more than call `super(config)`. Any expensive initialization, such as loading a manifest, should be handled by a separate singleton manager that the `test` method can query.

## Data Pipeline

The AssetValidator acts as a gatekeeper in the data flow from raw configuration files to live game objects. It intercepts string values and validates them before they are used to load assets from disk.

> Flow:
> Raw Config File (e.g., JSON) -> Hytale Codec Deserializer -> **Concrete AssetValidator Instance** -> Validation Result (boolean) -> Asset Builder or Game System

