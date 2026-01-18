---
description: Architectural reference for RequiredBlockFaceSupportValidator
---

# RequiredBlockFaceSupportValidator

**Package:** com.hypixel.hytale.server.core.asset.type.blocktype.config
**Type:** Singleton / Utility

## Definition
```java
// Signature
class RequiredBlockFaceSupportValidator implements LegacyValidator<Map<BlockFace, RequiredBlockFaceSupport[]>> {
```

## Architecture & Concepts
The RequiredBlockFaceSupportValidator is a specialized component within the server's asset validation pipeline. Its sole responsibility is to enforce logical correctness on the *support* configuration for block assets. During server startup or asset hot-reloading, block definition files are deserialized into memory. This validator intercepts the `support` map data structure to ensure content creators have not defined impossible, ambiguous, or redundant rules.

This class acts as a domain-specific rule engine for block physics and placement. By identifying configuration errors early, it prevents subtle and difficult-to-debug in-game behaviors, such as blocks that cannot be placed or blocks that defy expected structural integrity. It is a critical gatekeeper that guarantees the semantic validity of block assets before they are loaded into the main game state.

## Lifecycle & Ownership
- **Creation:** Instantiated as a static singleton field, `INSTANCE`, upon its first access by the Java ClassLoader. This is a zero-cost abstraction that requires no explicit management.
- **Scope:** Application-wide. A single instance exists for the entire lifetime of the server's JVM process.
- **Destruction:** The singleton instance is eligible for garbage collection only when its ClassLoader is unloaded, which typically occurs during a full server shutdown.

## Internal State & Concurrency
- **State:** Stateless. This class holds no mutable instance fields. Its `accept` method is a pure function whose output (the mutation of the `ValidationResults` object) depends solely on its input arguments.
- **Thread Safety:** Inherently thread-safe. As a stateless singleton, it can be safely invoked by multiple asset loading threads concurrently. The responsibility for managing the thread safety of the `ValidationResults` object lies with the calling system.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(support, results) | void | O(N) | Validates all block face support rules. N is the total number of support rules across all faces. Mutates the provided `ValidationResults` object with failures or warnings. |

## Integration Patterns

### Standard Usage
This validator is not intended for direct use in game logic. It is designed to be invoked by a higher-level asset deserializer or codec during the asset loading phase. The standard pattern is to retrieve the static instance and pass it the relevant data fragment and the results collector.

```java
// Hypothetical usage within an asset codec
ValidationResults results = new ValidationResults();
Map<BlockFace, RequiredBlockFaceSupport[]> supportRules = deserializedBlock.getSupportRules();

// Invoke the validator on the deserialized data
RequiredBlockFaceSupportValidator.INSTANCE.accept(supportRules, results);

if (results.hasFailed()) {
    throw new AssetLoadException("Block support validation failed: " + results.getFailures());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new RequiredBlockFaceSupportValidator()`. This violates the singleton pattern and provides no benefit, as the class is stateless. Always use the static `INSTANCE` field.
- **Ignoring Results:** The purpose of this class is to populate a `ValidationResults` object. Invoking `accept` and then failing to inspect the results object for failures or warnings renders the validation step useless and may allow corrupted assets into the game.

## Data Pipeline
This validator operates as a single, focused stage within the broader asset loading data pipeline. It receives a partially deserialized data structure and enriches a results object, which determines the fate of the asset.

> Flow:
> Raw Block Asset (.json) -> Hytale Codec System -> **RequiredBlockFaceSupportValidator** -> Populated ValidationResults -> Asset Manager Decision (Load/Reject)

