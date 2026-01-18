---
description: Architectural reference for StringTag
---

# StringTag

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.data
**Type:** Value Object

## Definition
```java
// Signature
public class StringTag implements CollectorTag {
```

## Architecture & Concepts
The StringTag class is an immutable wrapper for a primitive String, functioning as a type-safe identifier. Its primary architectural role is to represent a tag or label within the server's interaction and entity collection systems.

By encapsulating a raw string into a distinct StringTag type, the system avoids "primitive obsession," a common anti-pattern where primitive types like String are used to represent domain-specific concepts. This ensures that methods requiring a tag cannot accidentally be passed an arbitrary string, enhancing compile-time safety and code clarity.

StringTag instances are fundamental data containers, used to define criteria for entity selection and interaction logic. They are not services or managers; they are simple, inert data carriers whose identity is defined by their content, not their memory location.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively through the static factory method `StringTag.of(String)`. The constructor is private to enforce this pattern. This object is typically instantiated by configuration loaders or rule engines that parse interaction definitions from data files.
- **Scope:** A StringTag is a transient, short-lived object. It has no independent lifecycle and exists only as long as it is referenced by other components, such as a collection of interaction rules or a temporary query.
- **Destruction:** Management is delegated entirely to the Java Garbage Collector. As an immutable value object with no external resources, it requires no explicit cleanup.

## Internal State & Concurrency
- **State:** **Immutable**. The internal `tag` field is declared `final` and is assigned only once during construction. An instance of StringTag cannot be modified after it is created.
- **Thread Safety:** **Inherently thread-safe**. Due to its immutability, a StringTag instance can be freely shared and accessed across multiple threads without any risk of data corruption or race conditions. No external synchronization is required when using this class.

## API Surface
The public contract is minimal, focusing on creation and data retrieval. The class overrides `equals` and `hashCode` to enable correct behavior when used in collections like HashMaps or HashSets.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| of(String tag) | static StringTag | O(1) | **Factory Method.** The sole entry point for creating a StringTag instance. |
| getTag() | String | O(1) | Retrieves the underlying, immutable string value of the tag. |
| equals(Object o) | boolean | O(N) | Performs a value-based equality check. Returns true if the other object is a StringTag with an identical internal string. |
| hashCode() | int | O(N) | Computes a hash code based on the internal string, ensuring consistency with the equals method. |

## Integration Patterns

### Standard Usage
StringTag should be used as a type-safe identifier for querying or labeling entities and interactions. It is intended to be created from a configuration source and then used for comparison.

```java
// Example: Creating tags from a configuration source
StringTag enemyTag = StringTag.of("hostile_mob");
StringTag allyTag = StringTag.of("friendly_npc");

// Example: Using the tag in system logic to check an entity's properties
if (entity.hasTag(enemyTag)) {
    // Execute hostile-specific logic
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to instantiate with `new StringTag()`. This is prevented by the private constructor. Always use the `StringTag.of()` factory method. This pattern preserves the flexibility to add future optimizations like caching or interning without changing the public API.
- **Null Usage:** Avoid creating tags from null strings, as in `StringTag.of(null)`. While the class's `equals` and `hashCode` methods are null-safe, passing a null tag into the broader interaction system may result in a NullPointerException in downstream consumers that do not expect it.
- **High-Frequency Instantiation:** Avoid calling `StringTag.of()` repeatedly with the same string literal inside a performance-critical loop. While object creation is fast, it can create unnecessary GC pressure. For globally-used tags, it is best practice to create them once and store them as static final constants.

## Data Pipeline
StringTag acts as a data carrier within a larger configuration and logic pipeline. It represents the typed, in-memory form of a string defined in an external source.

> Flow:
> Configuration File (e.g., JSON) -> Server Deserializer -> **StringTag.of()** -> Interaction Rule Engine -> Entity Query System

