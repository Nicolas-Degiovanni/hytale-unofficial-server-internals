---
description: Architectural reference for ViewTest
---

# ViewTest

**Package:** com.hypixel.hytale.server.npc.util
**Type:** Utility / Enum

## Definition
```java
// Signature
public enum ViewTest implements Supplier<String> {
```

## Architecture & Concepts
ViewTest is a type-safe enumeration that defines the set of geometric algorithms available for determining an NPC's field of view. It serves as a foundational component within the server-side AI and visibility systems.

By encapsulating these distinct test types—NONE, VIEW_SECTOR, VIEW_CONE—the system avoids the use of "magic strings" or raw integers in configuration and logic. This prevents common errors related to typos or invalid values and ensures that only valid, known view test algorithms can be assigned to an NPC.

The implementation of the Supplier interface, specifically Supplier of String, provides a standardized way to retrieve a serializable, human-readable name for each test type. This is critical for integration with configuration files, debugging tools, and administrative command systems where a string representation is required.

## Lifecycle & Ownership
- **Creation:** ViewTest constants are instantiated automatically by the Java Virtual Machine (JVM) during the class-loading phase. Application code does not, and cannot, create instances of this enum.
- **Scope:** As with all enums, instances are static and persist for the entire lifetime of the application. They are effectively global, immutable singletons.
- **Destruction:** Instances are garbage collected only when the JVM shuts down.

## Internal State & Concurrency
- **State:** The internal state of each enum constant is immutable. The private final field *name* is assigned once during JVM instantiation and can never be changed.
- **Thread Safety:** This class is inherently thread-safe. Its immutability and the JVM's guarantee of single-instance creation eliminate the need for any external synchronization. It can be safely accessed and read from any thread at any time.

## API Surface
The primary API is the contract fulfilled from the Supplier interface, alongside the static methods provided to all Java enums.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | String | O(1) | Returns the serializable string name for the constant. |
| values() | ViewTest[] | O(N) | *Static method.* Returns a new array containing all enum constants. |
| valueOf(String) | ViewTest | O(N) | *Static method.* Parses a string to its corresponding enum constant. Throws IllegalArgumentException if no match is found. |

## Integration Patterns

### Standard Usage
ViewTest is intended to be used for identity comparison and in control flow statements like switch. Its string representation should be used for serialization or display purposes.

```java
// Retrieving the string value for configuration or logging
String testName = ViewTest.VIEW_CONE.get();

// Using in a conditional logic block
if (npc.getViewTest() == ViewTest.VIEW_SECTOR) {
    // Execute sector-based visibility logic
}

// Using in a switch statement for exhaustive checks
switch (npc.getViewTest()) {
    case VIEW_CONE:
        // ...
        break;
    case VIEW_SECTOR:
        // ...
        break;
    case NONE:
    default:
        // ...
        break;
}
```

### Anti-Patterns (Do NOT do this)
- **Comparison by String:** Never compare enum instances by their string value. This is inefficient and defeats the purpose of type safety.
  - **BAD:** `if (npc.getViewTest().get().equals("View_Cone"))`
  - **GOOD:** `if (npc.getViewTest() == ViewTest.VIEW_CONE)`
- **Reliance on Ordinal:** Do not use the `ordinal()` method for serialization or any persistent logic. The integer value can change if the order of constants in the source file is modified, leading to critical data corruption.
- **Null Checks:** An enum constant itself will never be null. A variable holding a reference to a ViewTest, however, can be. Always perform null checks on variables before use.

## Data Pipeline
ViewTest acts as a data type, not a data processor. It typically represents deserialized configuration that dictates the behavior of other systems.

> Flow:
> NPC Configuration File (e.g., JSON) -> Server Deserializer -> **ViewTest** instance -> NPC AI Behavior Tree -> Visibility Check Algorithm

