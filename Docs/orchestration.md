# Orchestration Services

An orchestration service is a fully configured agent runtime — a system prompt, a default LLM, a context store, tools, and an observer wired together and registered in DI.

## Creating an Orchestration Service

### Unnamed (default)

```csharp
builder.Services.AddOrchestrationService()
    .WithSystemPrompt("You are a helpful assistant.")
    .WithDefaultLlm("Anthropic")
    .WithContextStore();
```

Inject directly:

```csharp
public class MyController(IOrchestrationService orchestrator) { }
```

### Named

```csharp
builder.Services.AddOrchestrationService("Customer-Chat")
    .WithSystemPromptFile("Prompts/CustomerChat.md")
    .WithDefaultLlm("Anthropic")
    .WithContextStore<MyContextStore>()
    .WithObserver<ChatObserver>()
    .WithAgentTools();
```

Inject by name using .NET keyed services:

```csharp
public class ChatController(
    [FromKeyedServices("Customer-Chat")] IOrchestrationService orchestrator) { }
```

## Builder Methods

| Method | Description |
|---|---|
| `WithSystemPrompt(string)` | Set the system prompt inline |
| `WithSystemPromptFile(string)` | Load the system prompt from a file |
| `WithDefaultLlm(string)` | Set the default LLM provider by name |
| `WithContextStore()` | Use the default in-memory context store |
| `WithContextStore<T>()` | Use a custom `ISessionContextStore` implementation |
| `WithContextStore(factory)` | Use a factory to create the context store |
| `WithObserver<T>()` | Register an `IOrchestrationObserver` implementation |
| `WithObserver(factory)` | Use a factory to create the observer |
| `WithAgentTools()` | Include all registered tool services |
| `WithAgentToolset<T>()` | Include only a specific tool service class |
| `WithLlmOptions(configure)` | Configure LLM options (iterations, timeouts, model parameters) |
| `WithLlmResolver(factory)` | Provide a custom provider resolver |

## Sending Messages

### Standard (await the result)

```csharp
var result = await orchestrator.OrchestrateAsync(
    sessionId: "user-123",
    userMessage: "What's the weather in Seattle?",
    policy: null);

// result.FinalMessage      — the LLM's final text response
// result.Model             — which model was used
// result.TotalPromptTokens — prompt tokens across all iterations
// result.TotalCompletionTokens
// result.TotalToolCalls    — how many tools were called
// result.TotalIterations   — how many LLM round-trips
// result.HasError          — whether an error occurred
```

### Streaming

```csharp
await foreach (var evt in orchestrator.OrchestrateStreamAsync(
    sessionId: "user-123",
    userMessage: "What's the weather in Seattle?",
    policy: null))
{
    // Handle typed events as they arrive
}
```

See [Observers](observers.md) for the event types.

## Orchestration Policy

`OrchestrationPolicy` lets you override settings per-request without changing the service configuration:

```csharp
var policy = new OrchestrationPolicy
{
    LlmProvider = "Gemini",            // Override the default LLM
    SystemPrompt = "Be very concise.", // Override the system prompt

    // Pass request-scoped data to tools or observers
    RequestContext = new Dictionary<string, string>
    {
        ["userId"] = currentUser.Id,
        ["locale"] = "en-US"
    }
};

var result = await orchestrator.OrchestrateAsync(sessionId, message, policy);
```

## LLM Options

Configure orchestration behavior with `WithLlmOptions`:

```csharp
builder.Services.AddOrchestrationService("Search")
    .WithSystemPromptFile("Prompts/Search.md")
    .WithDefaultLlm("Gemini")
    .WithLlmOptions(options =>
    {
        options.MaxIterations = 5;              // Limit orchestration loop iterations
        options.EnableRateLimitWarnings = true;  // Log rate limit info
        options.ModelParameters = ModelParameters.DeterministicExtended;
    })
    .WithContextStore()
    .WithAgentToolset<SearchTools>();
```

### Model Parameter Presets

`ModelParameters` includes presets for common use cases:

| Preset | Description |
|---|---|
| `ModelParameters.Default` | Balanced defaults |
| `ModelParameters.Deterministic` | Low temperature, focused output |
| `ModelParameters.DeterministicExtended` | Deterministic with higher token limit |
| `ModelParameters.Concise` | Short, direct responses |
| `ModelParameters.Analytical` | Detailed, reasoning-oriented |
| `ModelParameters.Expressive` | Creative, varied output |

Or configure directly:

```csharp
options.ModelParameters = new ModelParameters
{
    Temperature = 0.3,
    MaxTokens = 4096,
    TopP = 0.9
};
```

## Thread Safety

Orchestration services handle concurrent requests safely. When multiple requests target the same session, built-in session locking ensures context integrity — no manual synchronization needed.

For direct context manipulation outside of orchestration, use `WithLockedContextAsync`:

```csharp
await orchestrator.WithLockedContextAsync(sessionId, async context =>
{
    // Safe to read/modify context here
    return ContextChangeResult.Changed;
});
```
