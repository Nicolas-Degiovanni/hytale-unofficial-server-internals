---
description: Architectural reference for OAuthFlow
---

# OAuthFlow

**Package:** com.hypixel.hytale.server.core.auth.oauth
**Type:** Transient State Object

## Definition
```java
// Signature
abstract class OAuthFlow {
```

## Architecture & Concepts
The OAuthFlow class is an abstract base class that serves as a state machine for a single, in-flight OAuth 2.0 authentication attempt. It is not a long-lived service, but rather a transient object designed to encapsulate the state and asynchronous result of one specific authentication flow.

Its primary architectural purpose is to decouple the initiating code from the asynchronous, callback-driven nature of a network-based authentication exchange. It achieves this by using a CompletableFuture as the public contract for the flow's outcome. Callers can initiate a flow, receive an OAuthFlow instance, and then use its future to block or chain subsequent actions that depend on the authentication result.

Concrete implementations of this class are expected to handle the specifics of a given OAuth grant type, such as the Authorization Code Flow or Client Credentials Flow. This base class provides the standardized state management and completion signaling mechanism for all such flows.

### Lifecycle & Ownership
- **Creation:** A new instance of a concrete OAuthFlow subclass is created for **every** authentication attempt. It is typically instantiated by a central authentication service or factory responsible for managing OAuth clients.
- **Scope:** Short-lived. The object's lifetime is strictly bound to the duration of a single authentication exchange, from initiation to the receipt of a success or failure callback.
- **Destruction:** The object is eligible for garbage collection as soon as its CompletableFuture is completed and no external references remain. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The OAuthFlow object is highly mutable. Its internal fields, such as tokenResponse, result, and errorMessage, are designed to be written to exactly once by an internal callback thread upon completion of the network request. The state transitions from an initial `UNKNOWN` result to either `SUCCESS` or `FAILED`.

- **Thread Safety:** This class is **not thread-safe** and must be handled with care in a multi-threaded environment. While the final state transition is protected from being triggered multiple times by the `isDone()` check on the future, the fields themselves are not protected by locks.

    **WARNING:** It is fundamentally unsafe for multiple threads to interact with an instance. The intended pattern is for a single network callback thread to call `onSuccess` or `onFailure`, and for one or more consumer threads to await the result solely through the `getFuture()` method. Directly accessing getters like `getResult()` before the future is complete constitutes a race condition.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onSuccess(TokenResponse) | final void | O(1) | Internal callback to finalize the flow as successful. Completes the future and sets the result state. Idempotent after the first call. |
| onFailure(String) | final void | O(1) | Internal callback to finalize the flow as failed. Sets the result state and error message. **WARNING:** This method does not complete the future. |
| getTokenResponse() | TokenResponse | O(1) | Retrieves the token response. Returns null if the flow did not succeed or has not completed. |
| getResult() | OAuthResult | O(1) | Retrieves the final result of the flow. Subject to race conditions if read before the future is complete. |
| getErrorMessage() | String | O(1) | Retrieves the error message. Returns null if the flow did not fail or has not completed. |
| getFuture() | CompletableFuture | O(1) | Returns the future that will be completed with the final OAuthResult. This is the primary, thread-safe mechanism for consuming the flow's outcome. |

## Integration Patterns

### Standard Usage
The intended usage is to obtain a concrete instance from a factory, initiate the flow (which happens outside this class), and use the future to handle the result asynchronously.

```java
// A hypothetical authentication service initiates the flow
ConcreteOAuthFlow flow = authService.beginAuthorizationCodeFlow();

// The consumer of the result uses the future to react to the outcome
flow.getFuture().thenAccept(result -> {
    if (result == OAuthResult.SUCCESS) {
        OAuthClient.TokenResponse token = flow.getTokenResponse();
        System.out.println("Authentication successful: " + token.getAccessToken());
    } else {
        System.err.println("Authentication failed: " + flow.getErrorMessage());
    }
});
```

### Anti-Patterns (Do NOT do this)
- **Instance Reuse:** Do not attempt to reuse an OAuthFlow instance for a second authentication attempt. Each instance represents a single, one-shot operation.
- **Direct Callback Invocation:** Never call `onSuccess` or `onFailure` from application code. These methods are strictly reserved for the underlying HTTP client's callback mechanism.
- **Polling State:** Do not poll `getResult()` in a loop to check for completion. This is inefficient and prone to race conditions. Always use the `CompletableFuture` returned by `getFuture()`.

## Data Pipeline
The OAuthFlow class acts as a stateful container within a larger control flow rather than a traditional data pipeline.

> Flow:
> Authentication Service Trigger -> **Concrete OAuthFlow Instantiation** -> Asynchronous Network Request -> Network Response -> `onSuccess`/`onFailure` Callback -> **OAuthFlow State Update** -> CompletableFuture Completion -> Consumer Thread Resumes Execution

