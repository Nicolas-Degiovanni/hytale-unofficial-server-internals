---
description: Architectural reference for RoleExistsValidator
---

# RoleExistsValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators.asset
**Type:** Utility / Flyweight

## Definition
```java
// Signature
public class RoleExistsValidator extends AssetValidator {
```

## Architecture & Concepts
The RoleExistsValidator is a concrete implementation of the **Strategy** pattern, specializing the generic AssetValidator contract. Its sole responsibility is to confirm that a given string identifier corresponds to a valid, registered NPC Role within the server's runtime.

This class acts as a critical guard within the server's asset loading and validation pipeline. By querying the NPCPlugin as the single source of truth for role definitions, it decouples asset parsing logic from the live state of the game's NPC system. This prevents the instantiation of NPC assets with invalid or misspelled roles, which would otherwise lead to severe runtime errors, undefined behavior, or difficult-to-diagnose bugs. It is a fundamental component for ensuring data integrity between static asset definitions and the dynamic server environment.

### Lifecycle & Ownership
-   **Creation:** The primary, default instance is created statically as a singleton during class loading. This shared instance is retrieved via the `required` factory method. Configured instances are created on-demand via the `withConfig` static factory method, typically by an asset building system during its own initialization.
-   **Scope:** The default singleton instance persists for the entire lifetime of the server JVM. Transient instances created with specific configurations are short-lived, their scope being confined to the asset builder that requested them.
-   **Destruction:** The singleton instance is eligible for garbage collection only upon server shutdown when its class loader is unloaded. Transient instances are garbage collected as soon as they are no longer referenced by the asset system.

## Internal State & Concurrency
-   **State:** Instances of RoleExistsValidator are effectively **immutable**. Any configuration is injected at construction time via the parent constructor and cannot be modified thereafter. The class does not cache role data; every validation check queries the NPCPlugin in real-time.
-   **Thread Safety:** This class is **conditionally thread-safe**. Its safety is entirely dependent on the thread safety of the `NPCPlugin.get().hasRoleName` method. Assuming the NPCPlugin's role registry is safe for concurrent reads (a standard design for registries populated at startup), this validator can be safely used across multiple threads, for example, in a parallelized asset loading system.

## API Surface
The public API is minimal, focusing on the validation contract and object creation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(String role) | boolean | O(1) | Executes the validation. Returns true if the role exists in the NPCPlugin registry. |
| errorMessage(String role, String attributeName) | String | O(1) | Generates a formatted, human-readable error message for validation failures. |
| required() | RoleExistsValidator | O(1) | Factory method to retrieve the shared, default singleton instance. |
| withConfig(EnumSet config) | RoleExistsValidator | O(1) | Factory method to create a new, transient instance with specific validator configurations. |

## Integration Patterns

### Standard Usage
This validator is not intended for direct use in general game logic. It is designed to be composed into a higher-level asset definition or builder system to enforce data integrity at load time.

```java
// Within an asset building or configuration system
// The builder uses the validator to enforce that the "role" field is valid.
AssetDefinitionBuilder npcBuilder = new AssetDefinitionBuilder();

npcBuilder.addAttribute(
    "role",
    "merchant",
    RoleExistsValidator.required()
);

// The builder's internal logic will invoke test("merchant")
// and throw a detailed exception if it returns false.
npcBuilder.validateAndBuild();
```

### Anti-Patterns (Do NOT do this)
-   **Dependency Inversion:** Do not attempt to use this validator for general-purpose role checks within game systems. It is designed for the asset pipeline. Game logic should query the NPCPlugin or a relevant role service directly.
-   **Ignoring Initialization Order:** The validator is critically dependent on the NPCPlugin being fully initialized and its roles loaded. Using this validator before the plugin is ready will result in false negatives for all checks. The asset validation stage must run after the NPC system bootstrap is complete.

## Data Pipeline
The RoleExistsValidator acts as a gate in the data flow from raw asset files to in-memory game objects. It does not transform data but rather asserts its validity, halting the pipeline on failure.

> Flow:
> NPC Asset File (e.g., JSON) -> Asset Deserializer -> Attribute Validation -> **RoleExistsValidator.test()** -> [**Success**] -> NPC Asset Instantiation
>
> **OR**
>
> NPC Asset File (e.g., JSON) -> Asset Deserializer -> Attribute Validation -> **RoleExistsValidator.test()** -> [**Failure**] -> ValidationException -> Error Log & Asset Load Abortion

