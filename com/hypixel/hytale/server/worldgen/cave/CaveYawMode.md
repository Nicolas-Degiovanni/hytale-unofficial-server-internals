---
description: Architectural reference for CaveYawMode
---

# CaveYawMode

**Package:** com.hypixel.hytale.server.worldgen.cave
**Type:** Utility (Type-safe Enum)

## Definition
```java
// Signature
public enum CaveYawMode {
   NODE,
   SUM,
   PREFAB;
   // ... method implementations
}
```

## Architecture & Concepts
CaveYawMode is a foundational enum within the server-side procedural world generation system, specifically for cave networks. It embodies the **Strategy design pattern** to dictate how the rotational orientation (yaw) of a new cave segment is calculated relative to its parent.

This component decouples the core cave generation algorithm from the specific rules of geometric orientation. By selecting a CaveYawMode, the generator can apply different behaviors for different types of cave segments. For example, a straight tunnel might inherit its parent's orientation (NODE), while a decorative side-chamber loaded from a prefab might need to add its own rotation (SUM) or completely override the parent's (PREFAB). This provides significant flexibility and control over the final structure of the cave system without complicating the primary generation loop.

### Lifecycle & Ownership
- **Creation:** Instances are created by the Java Virtual Machine during class loading. As an enum, its constants (NODE, SUM, PREFAB) are effectively static singletons. Direct instantiation is neither possible nor necessary.
- **Scope:** The enum constants exist for the entire lifetime of the server application. They are globally accessible and immutable.
- **Destruction:** Instances are garbage collected when the server process terminates and the class loader is unloaded.

## Internal State & Concurrency
- **State:** CaveYawMode is **stateless and immutable**. Each enum constant represents a pure, behavior-only strategy. It holds no internal data, and its operations produce results based solely on input arguments.
- **Thread Safety:** This class is unconditionally **thread-safe**. The combine method is a pure function without side effects, making it safe for concurrent use in multi-threaded world generation tasks without requiring any locks or synchronization.

## API Surface
The public contract consists of the three available strategy constants and the single abstract method they implement.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| NODE | CaveYawMode | O(1) | A strategy that ignores prefab rotation and inherits yaw directly from the parent node. |
| SUM | CaveYawMode | O(1) | A strategy that adds the prefab's yaw to the parent's yaw to produce a combined rotation. |
| PREFAB | CaveYawMode | O(1) | A strategy that completely overwrites the parent's yaw with the prefab's yaw. |
| combine(parentYaw, parentRotation) | float | O(1) | Executes the selected rotational strategy, returning the calculated yaw for a new cave segment. |

## Integration Patterns

### Standard Usage
This enum is intended to be used as a parameter or configuration value within a world generation function to select the desired orientation behavior.

```java
// Example within a hypothetical cave segment generator
void generateCaveSegment(float parentYaw, @Nullable PrefabRotation prefabRotation, CaveYawMode mode) {
    // The mode determines the final orientation of the new segment
    float newSegmentYaw = mode.combine(parentYaw, prefabRotation);

    // ... proceed to place the new cave segment at the calculated yaw
}

// Invocation
PrefabRotation chamberRotation = getChamberPrefabRotation();
generateCaveSegment(currentTunnelYaw, chamberRotation, CaveYawMode.SUM);
```

### Anti-Patterns (Do NOT do this)
- **Null-based Logic:** While the combine method is robust against a null PrefabRotation, do not rely on passing null to achieve the NODE behavior. If the intent is to inherit the parent yaw, explicitly use CaveYawMode.NODE for code clarity and to avoid accidental coupling with unrelated logic that might produce a null.
- **Switch Statements:** Avoid using a switch statement on a CaveYawMode instance to replicate its behavior. The entire purpose of the pattern is to call the combine method directly.

   **BAD:**
   ```java
   // Redundant and error-prone
   float result;
   switch (mode) {
       case NODE:
           result = parentYaw;
           break;
       // ... etc
   }
   ```

   **GOOD:**
   ```java
   // Correct and maintainable
   float result = mode.combine(parentYaw, prefabRotation);
   ```

## Data Pipeline
CaveYawMode acts as a functional transformation step within the larger cave generation data pipeline. It does not transport data but rather applies a rule to transform it.

> Flow:
> World Generator State -> Selects **CaveYawMode** Strategy -> `combine(parentYaw, prefabRotation)` -> New Segment Yaw -> Voxel Placement Stage

