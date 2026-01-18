---
description: Architectural reference for WeatherExistsValidator
---

# WeatherExistsValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators.asset
**Type:** Utility

## Definition
```java
// Signature
public class WeatherExistsValidator extends AssetValidator {
```

## Architecture & Concepts
The **WeatherExistsValidator** is a specialized component within the server's NPC asset-building framework. Its primary function is to enforce data integrity by ensuring that any weather reference within an NPC asset definition corresponds to an actual, loaded **Weather** asset in the central asset registry.

Architecturally, this class implements a Strategy pattern, where **AssetValidator** defines the contract for various validation strategies. This specific implementation acts as a guard, preventing the instantiation of NPCs with invalid or non-existent weather configurations, which could lead to runtime errors or unpredictable behavior in the game world. It serves as a critical link between the static data definitions of NPCs and the dynamic state of the engine's asset management system.

## Lifecycle & Ownership
-   **Creation:** Instances are not created directly via a public constructor. They are exclusively provisioned through two static factory methods:
    1.  **required()**: Returns a shared, static singleton instance (**DEFAULT_INSTANCE**). This is the most common and efficient way to obtain a validator.
    2.  **withConfig(config)**: Creates a new, transient instance with specific validation configurations. This is used for specialized cases where default validation behavior must be overridden.

-   **Scope:** The default singleton instance is application-scoped, created once at class-loading time and persisting for the entire server session. Configured instances are transient and their lifetime is tied to the scope of the asset-building operation that requested them.

-   **Destruction:** The singleton instance is reclaimed by the JVM during application shutdown. Transient instances are garbage collected once the asset validation process completes and they are no longer referenced.

## Internal State & Concurrency
-   **State:** The **WeatherExistsValidator** is effectively immutable. Its only potential state is an **EnumSet** of configuration flags inherited from **AssetValidator**, which is set at construction time and cannot be modified thereafter. The default singleton instance is stateless.

-   **Thread Safety:** This class is inherently thread-safe. It contains no mutable instance fields. The core validation logic in the **test** method performs a read-only operation against the static **Weather.getAssetMap()**. Assuming the underlying asset map is a thread-safe or effectively immutable collection after server initialization, instances of this validator can be safely shared and executed by multiple asset-building threads concurrently without synchronization.

## API Surface
The public contract is focused on providing the validation logic and constructing appropriate error messages.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(String value) | boolean | O(1) | The core validation method. Returns true if a **Weather** asset with the given name exists. |
| errorMessage(value, attribute) | String | O(1) | Generates a formatted, human-readable error message for a failed validation. |
| required() | WeatherExistsValidator | O(1) | Static factory. Returns the shared, default singleton instance of the validator. |
| withConfig(config) | WeatherExistsValidator | O(1) | Static factory. Returns a new validator instance with the specified configuration. |

## Integration Patterns

### Standard Usage
This validator is not intended for direct use in general game logic. It is designed to be plugged into a higher-level asset processing or validation system, typically during server startup or when live-reloading assets.

```java
// Hypothetical usage within an Asset Builder
// The builder would retrieve the appropriate validator for a "weather" field.
AssetValidator weatherRule = WeatherExistsValidator.required();

// The builder then tests a value parsed from an NPC definition file.
String npcWeatherValue = "sunny"; // from npc_definition.json
boolean isValid = weatherRule.test(npcWeatherValue);

if (!isValid) {
    String error = weatherRule.errorMessage(npcWeatherValue, "spawnCondition.weather");
    throw new AssetValidationException(error);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** The constructor is private. Do not attempt to create instances using reflection. Always use the static **required()** or **withConfig()** factory methods.
-   **Misuse for Other Asset Types:** This validator is hard-coded to check against the **Weather** asset registry. Using it to validate other asset types (e.g., items, sounds) will produce incorrect results. Use the appropriate validator for each asset domain.

## Data Pipeline
The **WeatherExistsValidator** operates as a single, synchronous step within a larger data validation pipeline for NPC assets.

> Flow:
> NPC Definition File (e.g., JSON) -> Server Asset Parser -> **WeatherExistsValidator**.test() -> Validation Result (Boolean) -> Asset Builder Logic

