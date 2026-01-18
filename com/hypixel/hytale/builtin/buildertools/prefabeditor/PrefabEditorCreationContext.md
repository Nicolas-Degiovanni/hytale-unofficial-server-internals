---
description: Architectural reference for PrefabEditorCreationContext
---

# PrefabEditorCreationContext

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface PrefabEditorCreationContext {
```

## Architecture & Concepts
The PrefabEditorCreationContext interface defines an immutable contract for specifying the parameters of a new prefab editing session. It acts as a formal data transfer object (DTO) that decouples the session initiator (e.g., a user interface, a command handler) from the core prefab editing service.

By depending on this interface rather than a concrete implementation, the prefab editing system can be invoked from various sources without being coupled to their specific details. The contract represents a complete, self-contained "request" to generate a world space for editing a set of prefabs. Its design enforces that all necessary configuration is provided upfront, preventing the editing service from entering an invalid or incomplete state.

This interface is a critical component of the builder tools, ensuring that complex world generation and prefab placement operations are predictable and consistently configured.

## Lifecycle & Ownership
As an interface, PrefabEditorCreationContext does not have a lifecycle of its own. The lifecycle pertains to the *objects that implement it*.

- **Creation:** An implementing object is instantiated on-demand by a system that needs to initiate a prefab editing session. This is typically done in response to a player action, such as executing a command or interacting with a UI. The creator is responsible for populating the context with all required parameters.
- **Scope:** The scope of a context object is strictly transient and operation-specific. It is created, passed as an argument to the relevant service method (e.g., `PrefabEditorService.createSession`), and is expected to be discarded once the session has been successfully initialized.
- **Destruction:** The context object is eligible for garbage collection as soon as the service method that consumes it completes. It should not be stored or referenced after the initial creation operation.

## Internal State & Concurrency
- **State:** The interface itself is stateless. Any class that implements it is designed to be an **immutable** data carrier. The contract consists solely of getter methods, strongly implying that the state should be set at creation time and never modified thereafter. This immutability guarantees that the parameters for a session cannot be altered during the creation process, which is essential for stability.
- **Thread Safety:** The contract is inherently thread-safe due to its immutable design. An instance of PrefabEditorCreationContext can be safely passed between threads without locks or synchronization, as its state is guaranteed not to change.

## API Surface
The API surface consists entirely of accessor methods to retrieve the configuration parameters for the prefab editing session.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getEditor() | Player | O(1) | Retrieves the player entity who initiated this editing session. |
| getEditorRef() | PlayerRef | O(1) | Returns a stable reference to the editor player. |
| getPrefabPaths() | List<Path> | O(1) | Provides the fully resolved file system paths to the prefab files to be loaded. |
| getBlocksBetweenEachPrefab() | int | O(1) | Specifies the empty block spacing to place between each loaded prefab. |
| getPasteLevelGoal() | int | O(1) | Defines the target Y-level (height) at which to paste the prefabs. |
| loadChildPrefabs() | boolean | O(1) | Determines if prefabs referenced within the primary prefabs should also be loaded. |
| shouldLoadEntities() | boolean | O(1) | Flag indicating whether entities stored within the prefabs should be spawned. |
| getStackingAxis() | PrefabStackingAxis | O(1) | Defines the primary axis (X, Y, or Z) along which to arrange multiple prefabs. |
| getWorldGenType() | WorldGenType | O(1) | Specifies the type of world to generate (e.g., flat, void). |
| getBlocksAboveSurface() | int | O(1) | For non-flat worlds, specifies the height above the generated surface to paste prefabs. |
| getAlignment() | PrefabAlignment | O(1) | Defines how prefabs are aligned relative to each other (e.g., center, corner). |
| getPrefabRootDirectory() | PrefabRootDirectory | O(1) | Specifies the root directory from which prefab paths are resolved. |
| isWorldTickingEnabled() | boolean | O(1) | Determines if the generated world should process game ticks (e.g., entity AI, block updates). |
| getRowSplitMode() | PrefabRowSplitMode | O(1) | Defines the logic for arranging prefabs into rows and columns. |
| getUnprocessedPrefabPaths() | List<String> | O(1) | Returns the raw, unprocessed prefab path strings as provided by the user. |
| getEnvironment() | String | O(1) | Specifies the environmental preset (e.g., lighting, skybox) for the world. |
| getGrassTint() | String | O(1) | Defines the color tint to apply to grass blocks in the generated world. |

## Integration Patterns

### Standard Usage
The context object should be constructed by a builder or factory, populated with user-defined settings, and passed to a service that consumes it. The caller should not retain a reference to the context object.

```java
// A hypothetical builder creates an object implementing the interface
PrefabEditorCreationContext context = new PrefabEditorContextBuilder()
    .editor(player)
    .addPrefab("my_awesome_build.prefab")
    .stackingAxis(PrefabStackingAxis.X)
    .worldGenType(WorldGenType.FLAT_WORLD)
    .build();

// The context is passed to the core service to initiate the operation
PrefabEditorService editorService = server.getService(PrefabEditorService.class);
editorService.createNewSession(context);
```

### Anti-Patterns (Do NOT do this)
- **Long-Lived References:** Do not store an instance of a PrefabEditorCreationContext in a field or cache. They are meant to be single-use, transient objects for a specific operation.
- **Casting and Modification:** Do not attempt to cast the context to its concrete implementation to modify its state after creation. The system relies on the immutability of these parameters once an operation has begun.
- **Manual Implementation:** Avoid creating one-off anonymous classes that implement this interface. Use designated builders or factories to ensure all parameters are handled correctly and to maintain forward compatibility.

## Data Pipeline
PrefabEditorCreationContext does not process data itself; it serves as the configuration payload that initiates a data pipeline for world generation and prefab loading.

> Flow:
> Player Input (Command/UI) -> Command Handler -> Concrete Context Builder -> **PrefabEditorCreationContext** (Immutable Instance) -> PrefabEditorService -> World Generation & Prefab Pasting Pipeline

