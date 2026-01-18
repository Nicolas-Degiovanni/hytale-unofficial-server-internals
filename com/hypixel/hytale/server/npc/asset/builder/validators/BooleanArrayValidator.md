---
description: Architectural reference for BooleanArrayValidator
---

# BooleanArrayValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility

## Definition
```java
// Signature
public abstract class BooleanArrayValidator extends Validator {
```

## Architecture & Concepts
The BooleanArrayValidator is an abstract base class that establishes a formal contract for validating boolean array properties within the server-side asset pipeline. It serves as a foundational component in a strategy pattern, allowing for interchangeable validation rules to be applied to NPC asset definitions during their construction.

This class is not used directly but is extended to create concrete validation logic, such as ensuring an array has a specific length or that at least one of its elements is true. Its primary role is to enforce data integrity for NPC configurations loaded from disk, preventing malformed or logically inconsistent assets from being loaded into the game server. It acts as a gatekeeper within the `NPCAssetBuilder`'s data processing flow.

### Lifecycle & Ownership
- **Creation:** This class is abstract and is never instantiated. Concrete subclasses are instantiated by a higher-level factory or builder, such as the `NPCAssetBuilder`, when it configures its validation chain for a specific asset property.
- **Scope:** Instances of concrete subclasses are transient. They are typically created for the duration of a single asset build process and are discarded once validation is complete.
- **Destruction:** Managed by the Java Garbage Collector. Once the asset builder that created the validator instance goes out of scope, the validator is eligible for cleanup. No explicit destruction logic is required.

## Internal State & Concurrency
- **State:** This class is stateless. It defines a contract for pure functions whose output depends solely on their input arguments. Concrete implementations are strongly expected to remain stateless.
- **Thread Safety:** The class is inherently thread-safe. As long as subclasses adhere to the stateless contract, a single instance of a concrete validator can be safely shared and executed across multiple threads without synchronization.

**WARNING:** Implementing a stateful subclass is a severe anti-pattern. It will break the assumption of reusability and introduce race conditions if the asset pipeline is ever parallelized.

## API Surface
The public API consists entirely of abstract methods that must be implemented by subclasses.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(boolean[] var1) | boolean | O(N) | Abstract method. Executes the core validation logic against the input array. Returns true if valid, false otherwise. |
| errorMessage(String var1, boolean[] var2) | String | O(N) | Abstract method. Generates a context-aware error message, typically including the name of the property being validated. |
| errorMessage(boolean[] var1) | String | O(N) | Abstract method. Generates a generic error message without property context. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly. Instead, they implement a concrete subclass defining a specific rule. The asset building framework then invokes this implementation.

```java
// 1. A concrete validator is defined
public class RequireAtLeastOneTrue extends BooleanArrayValidator {
    @Override
    public boolean test(boolean[] values) {
        for (boolean value : values) {
            if (value) {
                return true;
            }
        }
        return false;
    }

    @Override
    public String errorMessage(String propertyName, boolean[] values) {
        return "Property '" + propertyName + "' must have at least one true value.";
    }
    
    // ... other errorMessage implementation
}

// 2. The framework uses the validator (conceptual)
NPCAssetBuilder builder = new NPCAssetBuilder();
builder.validateBooleanField("interactionFlags", new RequireAtLeastOneTrue());
builder.build();
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to instantiate `new BooleanArrayValidator()`. This is an abstract class and will result in a compilation error.
- **Stateful Subclasses:** Do not add member variables to subclasses that store state between calls to the `test` method. Each validation must be an independent, idempotent operation.
- **Ignoring Error Messages:** Implementing the `test` method without providing a clear, corresponding implementation for `errorMessage` defeats the purpose of the class, as it will be impossible to debug validation failures.

## Data Pipeline
This validator is a single step within the broader NPC asset loading and validation pipeline.

> Flow:
> NPC Asset File (JSON/HOCON) -> HOCON Parser -> `NPCAssetBuilder` -> **BooleanArrayValidator** (Concrete Subclass) -> Validation Result (Pass/Fail)

