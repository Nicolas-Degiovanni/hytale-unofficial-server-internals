---
description: Architectural reference for EntityStatExistsValidator
---

# EntityStatExistsValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators.asset
**Type:** Utility

## Definition
```java
// Signature
public class EntityStatExistsValidator extends AssetValidator {
```

## Architecture & Concepts
The EntityStatExistsValidator is a specialized component within the server's asset validation framework. Its primary function is to enforce data integrity during the loading and parsing of game assets, specifically those related to Non-Player Characters (NPCs).

This class acts as a runtime guard clause. When an asset definition, such as a JSON or HOCON file for an NPC, references an entity statistic by name (e.g., "health", "attackDamage"), this validator is invoked to confirm that the referenced statistic is a known, registered type within the core game engine. It queries the central **EntityStatType** asset registry to perform this check.

By failing fast during the server's boot or asset-reload sequence, this validator prevents subtle and difficult-to-diagnose runtime errors that would otherwise occur if an NPC were spawned with an invalid or misspelled statistic. It is a critical link in the chain of ensuring that game design data is coherent and compatible with the engine's runtime systems.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively through the static factory methods **required()** or **withConfig(config)**. The **required()** method employs a Flyweight pattern, returning a shared, stateless default instance to minimize object allocation for the most common use case. The **withConfig(config)** method creates a new, transient instance tailored with specific validation behaviors.
- **Scope:** The lifecycle of an instance is typically ephemeral. It is created, used for a single validation operation or a batch of operations within an asset loading transaction, and then becomes eligible for garbage collection. The shared default instance, however, persists for the lifetime of the JVM.
- **Destruction:** Managed entirely by the Java Garbage Collector. No manual cleanup is required.

## Internal State & Concurrency
- **State:** This class is **immutable and stateless**. Its validation logic is deterministic and depends solely on its string input and the static, globally-accessible state of the **EntityStatType** asset map. Any configuration provided via the constructor is final.
- **Thread Safety:** This class is **thread-safe**. As it holds no mutable instance state, it can be safely shared and executed by multiple threads concurrently, such as in a parallelized asset loading system.

    **Warning:** The thread safety of this validator is contingent upon the thread safety of the underlying **EntityStatType.getAssetMap()**. It is assumed that this central asset map is populated during a single-threaded startup phase and subsequently treated as a read-only collection.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(String entityStat) | boolean | O(1) | Verifies if the given stat name exists in the central **EntityStatType** registry. |
| errorMessage(String, String) | String | O(1) | Generates a descriptive error message for a failed validation. |
| required() | EntityStatExistsValidator | O(1) | Returns the shared, default instance of the validator. |
| withConfig(EnumSet) | EntityStatExistsValidator | O(1) | Constructs a new validator instance with specific configuration flags. |

## Integration Patterns

### Standard Usage
This validator is intended to be used by higher-level asset builders or validation services. The typical pattern is to retrieve the shared instance and apply it as part of a larger validation chain when parsing an asset.

```java
// Within an asset building or parsing system
AssetValidator validator = EntityStatExistsValidator.required();
boolean isValid = validator.test("hytale:health");

if (!isValid) {
    throw new AssetParseException(validator.errorMessage("hytale:health", "baseStats"));
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructors are private. Do not attempt to create instances using reflection. Always use the static factory methods **required()** or **withConfig()**.
- **State Assumption:** Do not assume this class caches or holds the list of valid stats. It is a stateless pass-through validator that queries the authoritative **EntityStatType** registry on each call to **test()**. The source of truth for entity stats is external to this class.

## Data Pipeline
The validator sits at a critical checkpoint in the data flow from raw asset files to live game objects. It intercepts string identifiers and validates them against a central registry before they can be processed further.

> Flow:
> NPC Definition File (e.g., npc.json) -> Asset Deserializer -> String value ("hytale:mana") -> **EntityStatExistsValidator.test()** -> Boolean Result -> Asset Builder (Halts on false, proceeds on true) -> In-Memory Asset Object

