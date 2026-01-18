---
description: Architectural reference for RoleDebugFlags
---

# RoleDebugFlags

**Package:** com.hypixel.hytale.server.npc.role
**Type:** Utility

## Definition
```java
// Signature
public enum RoleDebugFlags implements Supplier<String> {
```

## Architecture & Concepts
The **RoleDebugFlags** enum provides a type-safe mechanism for enabling and disabling diagnostic features for the server-side Non-Player Character (NPC) AI system. It replaces traditional integer-based bitmasks with the more robust and readable **java.util.EnumSet**, preventing common errors associated with bitwise operations.

This class serves as a centralized registry for all available debugging options related to NPC roles, which encompass behaviors like pathfinding, motion control, collision detection, and state transitions. Its primary architectural function is to decouple the debug command-line interface from the individual AI subsystems. A command parser can use this class to translate raw string input into a strongly-typed configuration set, which is then passed to the relevant NPC entity.

A key concept is the **Preset**. A preset is a named, pre-configured collection of individual flags, such as *move* or *display*. This simplifies the user experience by providing shortcuts for common debugging scenarios, removing the need for developers or administrators to remember dozens of individual flag names.

## Lifecycle & Ownership
- **Creation:** As an enum, all flag instances (e.g., **TraceFail**, **Collisions**) are constructed and initialized by the JVM during class loading. They are static, final, and exist as singletons for the lifetime of the application. The internal **RoleDebugPreset** array is also initialized statically at class load time.
- **Scope:** The enum constants are global and persist for the entire server session. The **EnumSet** collections returned by static methods like **getFlags** are transient objects, owned by the caller that invoked the method. Their lifecycle is determined by the scope in which they are used, typically tied to an NPC's state for a specific debugging session.
- **Destruction:** Enum constants are eligible for garbage collection only when the defining ClassLoader is unloaded, which for server plugins typically means server shutdown. Transient **EnumSet** instances are garbage collected according to standard Java memory management rules once they are no longer referenced.

## Internal State & Concurrency
- **State:** The **RoleDebugFlags** enum is fundamentally immutable. Each enum constant holds a final **description** string. The static **presets** array is initialized once and is not modified at runtime. All public methods are pure functions that do not alter any internal static state; they produce new **EnumSet** instances based on their input.
- **Thread Safety:** This class is inherently thread-safe. Its immutable nature and lack of shared, mutable state eliminate the possibility of race conditions. Static methods can be safely invoked from any thread without external synchronization. The **EnumSet** instances it returns are not thread-safe by default, and callers are responsible for ensuring safe access if the set is shared between threads.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getFlags(String[] args) | EnumSet | O(N*P) | Parses an array of strings, resolving each to a flag or preset. Throws IllegalArgumentException on failure. N is number of args, P is number of presets. |
| getPreset(String arg) | EnumSet | O(P) | Retrieves a new EnumSet containing all flags for a single named preset. Throws IllegalArgumentException if the preset does not exist. P is number of presets. |
| havePreset(String name) | boolean | O(P) | Checks for the existence of a named preset without throwing an exception. P is number of presets. |
| getListOfAllFlags(StringBuilder) | StringBuilder | O(M) | Appends a comma-separated list of all defined flags to a StringBuilder. M is the total number of enum constants. |
| getListOfAllPresets(StringBuilder) | StringBuilder | O(P) | Appends a comma-separated list of all defined preset names to a StringBuilder. P is the number of presets. |

## Integration Patterns

### Standard Usage
This class is designed to be used by systems that process text-based commands, such as an in-game console or administrative tool. The primary entry point is the static **getFlags** method.

**Warning:** The parsing methods throw **IllegalArgumentException** for invalid input. Callers must implement proper exception handling to provide useful feedback to the user.

```java
// In a command handler that receives debug arguments
public void applyDebugFlags(Entity npc, String[] commandArgs) {
    try {
        EnumSet<RoleDebugFlags> flags = RoleDebugFlags.getFlags(commandArgs);
        
        // Attach the resulting flag set to a component on the NPC
        NpcDebugComponent debugComponent = npc.getComponent(NpcDebugComponent.class);
        if (debugComponent != null) {
            debugComponent.setDebugFlags(flags);
        }
    } catch (IllegalArgumentException e) {
        // Send feedback to the command issuer
        sendCommandFeedback("Error: " + e.getMessage());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Exceptions:** Failure to catch **IllegalArgumentException** from **getFlags** or **getPreset** will result in an unhandled exception, potentially crashing the command processing thread.
- **String Comparison:** Do not use `flag.name().equals("TraceFail")`. Use the type-safe enum constant directly: `flags.contains(RoleDebugFlags.TraceFail)`.
- **Relying on Ordinals:** Do not use the `ordinal()` method for serialization or logic. The order of enum constants is not guaranteed to be stable across code changes.

## Data Pipeline
The primary data flow for **RoleDebugFlags** is the transformation of raw user input into a structured, type-safe configuration object that can be consumed by various game systems.

> Flow:
> User Input (e.g., "/npc debug 12 steer display") -> Server Command Parser -> **RoleDebugFlags.getFlags** -> `EnumSet<RoleDebugFlags>` -> NpcDebugComponent (State) -> AI Subsystems (e.g., MotionController, Pathfinder) -> Conditional Logging/Visualization Output

