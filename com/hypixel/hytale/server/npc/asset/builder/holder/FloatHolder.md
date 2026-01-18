---
description: Architectural reference for FloatHolder
---

# FloatHolder

**Package:** com.hypixel.hytale.server.npc.asset.builder.holder
**Type:** Transient

## Definition
```java
// Signature
public class FloatHolder extends DoubleHolderBase {
```

## Architecture & Concepts
The FloatHolder is a specialized, transient data container used within the server's NPC asset building framework. It represents a value that will ultimately be resolved into a primitive float, but may be defined in a JSON configuration as a literal number, a string-based mathematical expression, or be omitted entirely to use a default.

Architecturally, it serves as a type-safe wrapper that decouples the asset parsing phase from the final value resolution phase. It inherits from DoubleHolderBase, following a design pattern where the base class handles the complex logic of JSON parsing, expression evaluation, and validation using 64-bit precision (double), while the concrete subclass provides the final type-specific casting and contract. This prevents code duplication across different numeric holders like IntegerHolder or DoubleHolder.

Its primary responsibility is to hold an unresolved value and provide a single method, get, to compute the final float value on-demand, using a given ExecutionContext.

## Lifecycle & Ownership
- **Creation:** A FloatHolder is instantiated dynamically by a higher-level asset builder (e.g., NpcAssetBuilder) when it encounters a field in the NPC's JSON definition that requires a float. The builder populates the instance by calling its readJSON method.
- **Scope:** The object's lifetime is strictly limited to the duration of a single NPC asset build process. It is a short-lived, intermediate object.
- **Destruction:** It becomes eligible for garbage collection as soon as the definitive float value has been retrieved via the get method and assigned to the final, persistent NPC asset object. It holds no references that would extend its life beyond the build operation.

## Internal State & Concurrency
- **State:** The FloatHolder is a mutable, stateful object. Its internal state, inherited from DoubleHolderBase, is populated by the readJSON method. This state contains the raw, unresolved value from the JSON source. The state is resolved into a primitive float only when the get method is invoked.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, configured, and used within the confines of a single, synchronous asset-building thread. The ExecutionContext parameter, which carries evaluation-specific state, makes concurrent calls to the get method inherently unsafe and will lead to unpredictable behavior. Do not share FloatHolder instances across threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readJSON(JsonElement, float, ...) | void | O(1) | Configures the holder's internal state from a JSON element. This method must be called exactly once before any other method. **Warning:** The provided implementation appears to be a recursive call which will cause a StackOverflowError; it is assumed the intended implementation calls the superclass method. |
| get(ExecutionContext) | float | O(E) | Resolves the stored expression or literal into a float value. E is the complexity of the expression evaluation. Throws exceptions if validation fails. |
| validate(ExecutionContext) | void | O(E) | A convenience method that triggers the same resolution and validation logic as get, used to verify an asset's integrity without needing the return value. |

## Integration Patterns

### Standard Usage
The FloatHolder is not intended for direct use by game logic developers. It is an internal component of the asset pipeline. A builder class is responsible for its entire lifecycle.

```java
// Conceptual example within an asset builder
// 1. A JsonElement is parsed from a config file.
JsonElement element = npcJson.get("health");

// 2. The builder creates and configures the holder.
FloatHolder healthHolder = new FloatHolder();
healthHolder.readJSON(element, 100.0f, healthValidator, "health", builderParams);

// 3. Later, during final asset construction...
ExecutionContext context = new ExecutionContext(...);
float finalHealth = healthHolder.get(context); // Resolve the value
npc.setHealth(finalHealth);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not manually create a FloatHolder using new without immediately configuring it via readJSON. An un-configured instance is in an invalid state and will fail on any subsequent method call.
- **State Mutation:** Do not call readJSON more than once on the same instance. These objects are designed to be configured once and are not reusable.
- **Concurrent Access:** Never pass a FloatHolder instance to another thread. The resolution process via get is stateful and not isolated.

## Data Pipeline
The FloatHolder acts as an intermediate stage in the data flow from raw configuration to a finalized game asset.

> Flow:
> JSON Configuration File -> JsonElement -> **FloatHolder.readJSON()** -> **FloatHolder.get(context)** -> Final float value -> NPC Game Object Field

