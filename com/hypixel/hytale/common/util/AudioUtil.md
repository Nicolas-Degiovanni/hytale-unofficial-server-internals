---
description: Architectural reference for AudioUtil
---

# AudioUtil

**Package:** com.hypixel.hytale.common.util
**Type:** Utility

## Definition
```java
// Signature
public class AudioUtil {
```

## Architecture & Concepts
AudioUtil is a stateless, low-level mathematical utility class that provides foundational calculations for the Hytale audio engine. It serves as the single source of truth for converting between perceptual audio units (decibels, semitones) and the linear values required by underlying audio APIs like OpenAL (linear gain, pitch multipliers).

This class is intentionally isolated from the rest of the engine. It has no dependencies and holds no state. Its primary architectural role is to decouple the game's audio logic (e.g., sound configuration, UI sliders) from the implementation details of the audio backend. By centralizing these critical formulas, it ensures consistency and accuracy in all audio-related calculations across the entire application. It is a foundational building block, not an active component in the engine's lifecycle.

## Lifecycle & Ownership
As a static utility class, AudioUtil has no instances and therefore no traditional object lifecycle.

- **Creation:** The class is loaded into the JVM by the ClassLoader when one of its static methods or fields is first accessed. It is never instantiated. The compiler enforces this by providing a default private constructor.
- **Scope:** The class and its static methods are available for the entire application lifetime after being loaded.
- **Destruction:** The class is unloaded from the JVM upon application termination. There is no manual cleanup or destruction process.

## Internal State & Concurrency
- **State:** AudioUtil is completely stateless. It contains only `public static final` constants and pure, static methods. It does not cache data or maintain any internal state between calls.
- **Thread Safety:** This class is inherently thread-safe. Its methods are pure functions, meaning their output depends solely on their input arguments, and they produce no side effects. It can be safely called from any thread, including the main game thread, audio thread, or UI thread, without any need for external synchronization or locks.

## API Surface
The public contract consists exclusively of static conversion functions.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decibelsToLinearGain(float) | float | O(1) | Converts a volume in decibels to a linear gain multiplier. Clamps at the minimum decibel threshold. |
| linearGainToDecibels(float) | float | O(1) | Converts a linear gain multiplier to a volume in decibels. Returns the minimum decibel value for non-positive input. |
| semitonesToLinearPitch(float) | float | O(1) | Converts a pitch shift in semitones to a linear pitch multiplier. |
| linearPitchToSemitones(float) | float | O(1) | Converts a linear pitch multiplier to a pitch shift in semitones. |

## Integration Patterns

### Standard Usage
This class should be used whenever a conversion between perceptual and linear audio units is required. It is invoked directly via its static methods.

```java
// Example: Setting a sound's volume from a UI slider value in decibels
float volumeInDb = uiSlider.getValue(); // e.g., -6.0f
float linearGain = AudioUtil.decibelsToLinearGain(volumeInDb);

// The audio engine would then use this linear gain value
audioEngine.setSourceGain(soundId, linearGain);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create an instance with `new AudioUtil()`. This is impossible and indicates a misunderstanding of the class's purpose.
- **Formula Duplication:** Do not re-implement these conversion formulas elsewhere in the codebase. This would defeat the purpose of the utility and introduce potential for inconsistencies. All audio unit conversions must flow through this class.

## Data Pipeline
AudioUtil is not a component *in* a data pipeline; it is a stateless tool used *by* components within the pipeline to transform data. It acts as a pure function that is called on-demand.

> **Flow:**
> Game Logic (e.g., "Set volume to -20dB") -> AudioEngine -> **AudioUtil.decibelsToLinearGain(-20.0f)** -> Low-Level Audio API (e.g., OpenAL `alSourcef`)

