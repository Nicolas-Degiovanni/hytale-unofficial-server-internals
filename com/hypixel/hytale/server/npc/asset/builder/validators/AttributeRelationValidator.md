---
description: Architectural reference for AttributeRelationValidator
---

# AttributeRelationValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Transient

## Definition
```java
// Signature
public class AttributeRelationValidator extends Validator {
```

## Architecture & Concepts
The AttributeRelationValidator is a concrete implementation of the Strategy pattern, designed to enforce a specific, configurable rule within the server's NPC asset validation pipeline. Its sole responsibility is to verify that a predefined logical relationship, such as *GREATER THAN* or *EQUAL TO*, holds true between two named attributes of an NPC.

This class operates as a self-contained, immutable rule object. It is constructed by a higher-level system, typically an asset builder or parser, which reads NPC definition files. By encapsulating a single validation constraint, multiple AttributeRelationValidator instances can be composed into a larger ruleset, allowing for complex and flexible validation logic without hard-coding rules into the core asset loading system.

The use of string identifiers for attributes indicates that this validator is intended to operate on dynamic data structures, such as property maps, where attribute values are resolved by name at runtime during the validation phase.

### Lifecycle & Ownership
-   **Creation:** Instantiation is strictly controlled via the static factory method `withAttributes`. This is typically invoked by an asset parsing service that translates a declarative configuration (e.g., from a JSON file) into a collection of executable Validator objects.
-   **Scope:** Short-lived. An instance of AttributeRelationValidator is created to represent a single rule for a single asset validation pass. It holds no state beyond its initial configuration and is immediately eligible for garbage collection after the validation process completes.
-   **Destruction:** Handled entirely by the Java Garbage Collector. This class manages no native resources or persistent connections, requiring no explicit cleanup.

## Internal State & Concurrency
-   **State:** **Immutable**. All internal fields, including the attribute names and the relational operator, are declared `final` and are set exclusively at construction time. The object's configuration cannot be altered after it has been created.
-   **Thread Safety:** **Inherently thread-safe**. Its immutable nature guarantees that a single instance can be safely shared and executed across multiple threads without any risk of data corruption or race conditions. No external locking or synchronization is required.

## API Surface
The public contract is minimal, focused entirely on instantiation. The core validation logic is invoked through the parent `Validator` interface, which is not detailed here.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| withAttributes(String, RelationalOperator, String) | AttributeRelationValidator | O(1) | Static factory method. Constructs and returns a new, immutable validator instance configured with the specified rule. |

## Integration Patterns

### Standard Usage
This validator is not intended for direct, ad-hoc use. It is designed to be created and managed by a validation framework or an asset builder as part of a larger ruleset.

```java
// Example: An asset builder constructs a list of validation rules
// from an NPC's definition file.
List<Validator> rules = new ArrayList<>();

// Rule: Ensure the NPC's 'strength' attribute is greater than its 'dexterity'.
Validator strengthCheck = AttributeRelationValidator.withAttributes(
    "strength",
    RelationalOperator.GREATER_THAN,
    "dexterity"
);

rules.add(strengthCheck);

// A validation engine would later iterate through this list and
// execute each validator against the NPC's data.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** The constructor is `private` by design. Attempting to instantiate with `new AttributeRelationValidator()` will result in a compile-time error. Always use the `withAttributes` static factory method.
-   **Rule Modification:** Do not design systems that attempt to modify a validator's rule after creation. If a different rule is needed, discard the existing instance and create a new one with the desired configuration.

## Data Pipeline
The AttributeRelationValidator acts as a single, discrete step within a larger data validation pipeline. It does not manage data flow itself but rather acts upon data provided to it.

> Flow:
> NPC Asset Definition (File) -> Asset Builder -> **AttributeRelationValidator Instance** -> Validation Engine -> Validation Result (Pass/Fail)

