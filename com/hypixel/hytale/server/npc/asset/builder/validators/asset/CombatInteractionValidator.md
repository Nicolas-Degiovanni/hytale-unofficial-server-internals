---
description: Architectural reference for CombatInteractionValidator
---

# CombatInteractionValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators.asset
**Type:** Transient

## Definition
```java
// Signature
public class CombatInteractionValidator extends AssetValidator {
```

## Architecture & Concepts
The CombatInteractionValidator is a specialized rule-enforcement component within the server's asset processing pipeline. Its primary function is to act as a gatekeeper, ensuring that an `Interaction` asset designated for an NPC's combat behavior conforms to a strict, non-negotiable contract.

This validator decouples the complex, general-purpose Interaction system from the specific requirements of the Combat system. It prevents designers from inadvertently assigning game-breaking or logically inconsistent interactions—such as block-placing or terrain modification—to an NPC's attack routine.

Architecturally, it operates by performing a multi-faceted analysis on a target `RootInteraction` asset. This involves:
1.  **Tag Verification:** Checking for the presence and exclusivity of critical combat tags (e.g., Attack, Melee, Ranged).
2.  **Graph Traversal:** Walking the entire chain of linked interactions stemming from the root.
3.  **Type Disqualification:** Identifying and rejecting any interaction within the chain that belongs to a prohibited class.

This component is critical for maintaining server stability and predictable NPC behavior, enforcing design constraints at the data level before an NPC is ever instantiated in the world.

### Lifecycle & Ownership
-   **Creation:** Instantiated on-demand via the static factory methods `required()` or `withConfig()`. It is typically created by a higher-level asset builder or configuration loader during the validation phase of an NPC definition file.
-   **Scope:** Ephemeral. The object's lifetime is intended to be extremely short, confined to a single validation operation.
-   **Destruction:** The instance holds no external references and is immediately eligible for garbage collection after its `test` and `errorMessage` methods have been invoked.

## Internal State & Concurrency
-   **State:** The validator is stateful but its state is transient. Internal fields such as `assetExists`, `attackTag`, and `disallowedInteractions` are mutated during the execution of the `test` method. This state is specific to the last validation performed and is completely reset on the next call to `test`.

-   **Thread Safety:** **This class is not thread-safe.** Its mutable internal state is modified without any synchronization mechanisms. Sharing a single instance across multiple threads will lead to race conditions and unpredictable validation results.

    **WARNING:** A new instance of CombatInteractionValidator must be created for each distinct validation task. Do not cache or reuse instances across threads.

## API Surface
The public contract is minimal, focusing exclusively on the validation operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(String value) | boolean | O(N) | Executes the validation logic against the asset ID. N is the number of interactions in the chain. Populates internal state for error reporting. |
| errorMessage(String value, String attribute) | String | O(1) | Constructs a detailed, human-readable error message based on the internal state from the last `test` call. |
| required() | CombatInteractionValidator | O(1) | Static factory for creating a default validator instance. |
| withConfig(EnumSet) | CombatInteractionValidator | O(1) | Static factory for creating a configured validator instance. |

## Integration Patterns

### Standard Usage
The validator is designed to be used in a simple, linear sequence within a broader asset loading or building process.

```java
// In a hypothetical NPC asset builder
CombatInteractionValidator validator = CombatInteractionValidator.required();
String interactionAssetId = "npc.grob.basic_attack";

if (!validator.test(interactionAssetId)) {
    String error = validator.errorMessage(interactionAssetId, "primaryAttack");
    throw new AssetValidationException(error);
}

// If validation passes, proceed with asset assignment
npc.setPrimaryAttack(interactionAssetId);
```

### Anti-Patterns (Do NOT do this)
-   **State Reuse:** Do not call `errorMessage` without first calling `test` in the same operational scope. The error message is dependent on the state populated by the `test` method and will be invalid or stale otherwise.
    ```java
    // BAD: The state from the first test is overwritten
    validator.test("asset_one");
    validator.test("asset_two");
    String error = validator.errorMessage("asset_one", "attribute"); // Reports on asset_two!
    ```
-   **Concurrent Access:** Do not share an instance across threads. This will cause data corruption of the internal validation state.
    ```java
    // BAD: Race condition on internal fields
    CombatInteractionValidator sharedValidator = CombatInteractionValidator.required();
    thread1.run(() -> sharedValidator.test("asset_one"));
    thread2.run(() -> sharedValidator.test("asset_two"));
    ```

## Data Pipeline
The validator processes a string asset identifier and, through a series of lookups and traversals, reduces it to a boolean outcome.

> Flow:
> NPC Asset Definition (String ID) -> **CombatInteractionValidator.test()** -> RootInteraction.getAssetMap() -> InteractionManager.walkChain() -> **[Internal State Mutation]** -> Boolean Result

