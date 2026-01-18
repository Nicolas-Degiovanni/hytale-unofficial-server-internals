---
description: Architectural reference for OAuthResult
---

# OAuthResult

**Package:** com.hypixel.hytale.server.core.auth.oauth
**Type:** Type-Safe Enumeration

## Definition
```java
// Signature
public enum OAuthResult {
```

## Architecture & Concepts
The OAuthResult enum is a fundamental component of the server-side authentication subsystem. It serves as a standardized, type-safe contract for representing the outcome of an OAuth authentication attempt.

Its primary architectural role is to eliminate ambiguity and prevent the use of "magic values" such as integers or strings to represent success or failure states. By providing a fixed set of constants (SUCCESS, FAILED, UNKNOWN), it ensures that all components within the authentication pipeline—from the initial token validation service to the final session manager—communicate outcomes in a consistent and robust manner. This design choice significantly improves code clarity and reduces the risk of runtime errors caused by typos or misinterpreted state values.

## Lifecycle & Ownership
- **Creation:** Instances of this enum (UNKNOWN, SUCCESS, FAILED) are constructed by the Java Virtual Machine during class loading. They are compile-time constants and are not instantiated dynamically at runtime.
- **Scope:** Application-level. The enum constants exist for the entire lifetime of the server process.
- **Destruction:** The instances are managed by the JVM and are reclaimed only upon application shutdown. There is no manual memory management or destruction process.

## Internal State & Concurrency
- **State:** Inherently **immutable**. The state of an enum constant is fixed at compile time and cannot be altered.
- **Thread Safety:** This type is unconditionally **thread-safe**. As immutable, globally accessible constants, instances of OAuthResult can be safely passed between, stored, and read by multiple threads without any form of synchronization or locking.

## API Surface
The public contract consists of the defined enumeration constants.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| UNKNOWN | OAuthResult | O(1) | Represents an indeterminate or unhandled authentication outcome. |
| SUCCESS | OAuthResult | O(1) | Represents a successful authentication and validation of credentials. |
| FAILED | OAuthResult | O(1) | Represents a definitive authentication failure due to invalid credentials, expired tokens, or other errors. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to use OAuthResult as the return type for authentication methods and as the subject of a switch statement for handling different outcomes.

```java
// How a developer should normally use this
OAuthService authService = context.getService(OAuthService.class);
OAuthResult result = authService.validateToken(playerToken);

switch (result) {
    case SUCCESS:
        sessionManager.grantAccess(player);
        break;
    case FAILED:
        connection.disconnect("Authentication Failed.");
        break;
    case UNKNOWN:
    default:
        connection.disconnect("An unknown authentication error occurred.");
        break;
}
```

### Anti-Patterns (Do NOT do this)
- **Ordinal Comparison:** Do not rely on the `ordinal()` method for business logic (e.g., `if (result.ordinal() == 1)`). The numeric order of enum constants is an implementation detail and can change, leading to fragile code.
- **String Comparison:** Avoid comparing the result of `toString()` to a string literal (e.g., `if (result.toString().equals("SUCCESS"))`). Use direct object comparison (`result == OAuthResult.SUCCESS`) for performance and type safety.

## Data Pipeline
OAuthResult is typically the terminal output of an authentication process. It represents the final, resolved state of a complex series of validation steps.

> Flow:
> External OAuth Provider Response -> TokenValidationService -> **OAuthResult** -> SessionCreationManager -> Player Connection State Update

