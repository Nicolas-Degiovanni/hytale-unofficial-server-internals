---
description: Architectural reference for UIButton
---

# UIButton

**Package:** com.hypixel.hytale.codec.schema.metadata.ui
**Type:** Data Schema / DTO

## Definition
```java
// Signature
public class UIButton {
```

## Architecture & Concepts
The UIButton class is a data schema, not a functional UI component. It serves as a Plain Old Java Object (POJO) that represents the serialized state of a button as defined in UI layout files or network packets. Its primary role is to act as a structured container for data that is being deserialized by the engine's codec system.

This class is a critical part of the data-driven UI framework. It decouples the raw data representation (e.g., a JSON object) from the runtime UI implementation. The static CODEC field is the most significant feature, defining the contract for how raw data is mapped to this Java object. The engine uses this codec to instantiate and populate UIButton objects during asset loading or UI synchronization.

**Warning:** This object does not contain rendering logic, state (e.g., hovered, pressed), or event handlers. It is purely a data-transfer object used during the UI inflation process.

## Lifecycle & Ownership
- **Creation:** UIButton instances are created almost exclusively by the Hytale codec framework. When a UI asset is parsed, the static `UIButton.CODEC` is invoked, which in turn calls the protected no-argument constructor and populates the fields. Manual instantiation via the public constructor is rare and should only be used for programmatic UI generation.
- **Scope:** The object is **transient and short-lived**. Its lifecycle is typically confined to a single frame or a single method scope during the UI building phase. It exists only long enough for its data to be consumed by a UI factory or layout engine, which then creates the actual, stateful UI widget.
- **Destruction:** The object is eligible for garbage collection immediately after its data has been transferred to a corresponding runtime UI component. Holding long-term references to UIButton instances is a memory leak anti-pattern.

## Internal State & Concurrency
- **State:** The internal state is **mutable** during deserialization. The `textId` and `buttonId` fields are populated by the codec after the object is constructed. After population, the object should be treated as immutable by consumers. There are no public setters to enforce this, so discipline is required.
- **Thread Safety:** This class is **not thread-safe**. It is a simple data container with no synchronization mechanisms. It is designed to be created, populated, and read by a single thread, typically the main game thread or a dedicated asset loading thread.

**Warning:** Never share an instance of UIButton across multiple threads. Do not read its fields from one thread while another thread (e.g., the codec system) might be writing to them.

## API Surface
The public API is intentionally minimal, exposing only the constructor and the static codec for the serialization system. Field access is managed internally by the codec.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| UIButton(textId, buttonId) | constructor | O(1) | Manually constructs a UIButton. Primarily for testing or dynamic UI code. |
| CODEC | BuilderCodec | N/A | Static constant used by the engine to serialize and deserialize UIButton instances. |

## Integration Patterns

### Standard Usage
Developers will almost never interact with this class directly. The engine's UI system uses its codec to inflate layouts defined in data files.

```java
// Engine-level code (conceptual)
// The codec is used to decode a data source into a UIButton object.
DataSource source = loadUiFile("main_menu.json");
UIButton buttonData = UIButton.CODEC.decode(source);

// The data is then used by a factory to create a real UI widget.
engine.getUIFactory().createButtonFromData(buttonData);
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not store instances of UIButton in caches or as member variables of long-lived services. They are transient data carriers.
- **State Modification:** Do not attempt to modify the fields of a UIButton after it has been created by the codec. This can lead to unpredictable UI behavior.
- **Direct Instantiation for Layouts:** While possible, avoid using `new UIButton()` to build complex UIs in code. The system is designed for data-driven layouts defined in external files.

## Data Pipeline
UIButton is an intermediate representation in the UI loading pipeline. It translates raw, untyped data into a structured Java object before the final, interactive UI element is created.

> Flow:
> UI Asset File (JSON/Binary) -> Byte Stream -> Hytale Codec Engine -> **UIButton instance** -> UI Layout Engine -> Renderable Widget

