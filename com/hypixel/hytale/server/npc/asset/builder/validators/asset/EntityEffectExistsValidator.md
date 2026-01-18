---
description: Architectural reference for EntityEffectExistsValidator
---

# EntityEffectExistsValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators.asset
**Type:** Utility (Singleton-pattern for default)

## Definition
```java
// Signature
public class EntityEffectExistsValidator extends AssetValidator {
```

## Architecture & Concepts
The EntityEffectExistsValidator is a concrete implementation of the **Strategy Pattern**, specializing the generic AssetValidator contract. Its primary function is to enforce data integrity within the server's asset pipeline, specifically for configurations that reference entity effects by name.

This validator acts as a gatekeeper during the deserialization and construction of higher-level assets, such as NPC behaviors or weapon definitions. It guarantees that any string identifier meant to represent an EntityEffect (e.g., "hytale:poison_cloud") corresponds to an actual, loaded asset in the central `EntityEffect` asset registry.

By performing this check at asset load time, the system prevents invalid configurations from entering the runtime environment. This preemptively eliminates a class of difficult-to-debug runtime errors, such as NullPointerExceptions, that would otherwise occur when the game logic attempts to resolve a non-existent effect.

### Lifecycle & Ownership
-   **Creation:** The default, configuration-free instance is created statically during class loading and exposed via the `required()` factory method. Configured instances are created on-demand by the asset building framework through the `withConfig()` factory method. Direct instantiation is prohibited by private constructors.
-   **Scope:** The default singleton instance is application-scoped, persisting for the entire server session. Configured instances are transient and their lifetime is bound to the specific asset validation operation for which they were created.
-   **Destruction:** The default instance is destroyed upon JVM shutdown. Transient instances are eligible for garbage collection as soon as the asset builder releases its reference to them.

## Internal State & Concurrency
-   **State:** This class is effectively immutable and stateless. The default instance holds no state. Configured instances hold a final, immutable `EnumSet` of configuration flags. The core validation logic is a pure function that queries an external, static asset map without causing side effects or modifying internal state.
-   **Thread Safety:** The class is inherently thread-safe. Its stateless nature and reliance on read-only operations against the central asset registry allow instances to be safely shared and executed by multiple asset-loading threads concurrently without locks or synchronization.

## API Surface
The public contract is minimal, focusing on the validation test and factory methods for instantiation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(String effect) | boolean | O(1) | **Core validation method.** Returns true if an EntityEffect with the given name exists in the global asset map. |
| required() | EntityEffectExistsValidator | O(1) | Static factory. Returns the shared, default instance of the validator. |
| withConfig(config) | EntityEffectExistsValidator | O(1) | Static factory. Returns a new, transient instance with specific validation configurations. |
| errorMessage(effect, attribute) | String | O(1) | Generates a formatted, human-readable error message for validation failures. |

## Integration Patterns

### Standard Usage
This validator is not intended for direct use in game logic. It is designed to be plugged into a higher-level asset building or validation framework. The framework is responsible for invoking the `test` method.

```java
// Hypothetical usage within an NPC asset builder
// The builder uses the validator to check the "onDeathEffect" field.

AttributeValidatorChain validators = new AttributeValidatorChain();
validators.add("onDeathEffect", EntityEffectExistsValidator.required());

// The framework would later iterate through validators and execute:
// boolean isValid = validator.test("hytale:dissolve_effect");
```

### Anti-Patterns (Do NOT do this)
-   **Bypassing Factories:** Attempting to instantiate this class via reflection is an anti-pattern. Always use the static `required()` or `withConfig()` methods to ensure predictable behavior.
-   **Incorrect Domain:** This validator is hardcoded to check the `EntityEffect` asset domain. Using it to validate other asset types (e.g., Sounds, Models) will produce incorrect results and bypass the intended validation for that type.
-   **Post-Load Validation:** Relying on this validator after the server has fully initialized is a misuse of its purpose. Its role is to ensure integrity *during* the asset loading phase, not to perform runtime checks.

## Data Pipeline
The EntityEffectExistsValidator is a critical checkpoint in the data pipeline that transforms raw configuration files into live game objects. It does not transform data but rather validates it.

> Flow:
> NPC Asset File (e.g., JSON) -> Asset Deserializer -> **EntityEffectExistsValidator** -> Validated Asset Data -> NPC Factory -> Live NPC in Game World

