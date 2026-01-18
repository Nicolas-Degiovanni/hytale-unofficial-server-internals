---
description: Architectural reference for ParticleSystemExistsValidator
---

# ParticleSystemExistsValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators.asset
**Type:** Utility (Stateless Validator)

## Definition
```java
// Signature
public class ParticleSystemExistsValidator extends AssetValidator {
```

## Architecture & Concepts
The ParticleSystemExistsValidator is a specific implementation of the generic AssetValidator contract. Its sole responsibility is to confirm that a string identifier corresponds to a particle system asset that has been successfully loaded into the engine's central asset registry.

This class operates as a validation rule within a larger data processing or asset building pipeline, most likely during server startup or when loading NPC definitions. By encapsulating this single piece of logic, it decouples the asset definition parser from the state of the global ParticleSystem asset map. This allows for a composable system where various validation rules can be attached to different asset attributes without modifying the core parsing logic.

The validation is performed by querying the static asset map provided by the ParticleSystem class. A successful validation indicates that the asset exists and is available for use by other server systems.

### Lifecycle & Ownership
-   **Creation:** The primary instance, DEFAULT_INSTANCE, is a singleton created during class loading. Additional, configured instances are created on-demand via the static factory method withConfig. These factories are the sole entry points for instantiation, as the constructors are private.
-   **Scope:** The DEFAULT_INSTANCE is application-scoped and persists for the entire server lifetime. Instances created via withConfig are transient and their lifetime is typically bound to the configuration or builder object that requested them.
-   **Destruction:** The singleton instance is eligible for garbage collection only upon JVM shutdown. Transient instances are garbage collected once they are no longer referenced by the asset building framework.

## Internal State & Concurrency
-   **State:** This class is effectively **immutable**. An instance may hold a configuration EnumSet provided at creation, but this state does not change during its lifetime. The validator itself holds no mutable state or caches. Its behavior is entirely dependent on the external state of the global ParticleSystem asset map.

-   **Thread Safety:** The validator is **conditionally thread-safe**. Its own internal structure is immutable and safe for concurrent access. However, its correctness relies entirely on the thread safety of the underlying global asset registry, accessed via ParticleSystem.getAssetMap(). It is assumed that the asset map is safe for concurrent reads after the initial asset loading phase is complete.

    **Warning:** Invoking this validator from a separate thread while the primary asset loading thread is still populating the asset map will lead to non-deterministic race conditions. Validation should only occur after all required assets are loaded.

## API Surface
The public contract is minimal, focusing on the validation logic and factory methods for instantiation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(String particleSystem) | boolean | O(1) avg | Executes the validation. Returns true if the particle system name exists in the global asset map. |
| errorMessage(String, String) | String | O(1) | Generates a formatted, human-readable error message for failed validations. |
| required() | ParticleSystemExistsValidator | O(1) | Static factory method to retrieve the shared, default singleton instance. |
| withConfig(EnumSet) | ParticleSystemExistsValidator | O(1) | Static factory method to create a new validator instance with specific configuration. |

## Integration Patterns

### Standard Usage
This validator is not intended for direct invocation in general game logic. It should be supplied to a builder or configuration system responsible for parsing and validating asset definitions at load time.

```java
// Hypothetical usage within an NPC asset builder
NpcAttributeBuilder attributeBuilder = new NpcAttributeBuilder("onHitEffect");

// The validator is attached as a rule to the builder
attributeBuilder.addValidator(ParticleSystemExistsValidator.required());

// The builder later uses the validator to check input data
ValidationResult result = attributeBuilder.validate("hytale:effects.particles.blood_spurt");
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not attempt to instantiate this class with reflection. Always use the static factory methods required() or withConfig().
-   **Runtime Validation:** Do not use this validator to check for particle system existence in the main game loop or during performance-critical operations. Asset validation is a load-time concern. Repeatedly querying the asset map via this abstraction is inefficient and semantically incorrect for runtime logic.

## Data Pipeline
The validator acts as a gate in the data flow from a raw asset definition to a validated, in-memory game object.

> Flow:
> Raw Asset Data (e.g., JSON file) -> Deserializer -> String value ("my_particle") -> **ParticleSystemExistsValidator.test()** -> Boolean Result -> Asset Builder Logic

