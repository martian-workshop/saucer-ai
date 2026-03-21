# Provider Configuration

Saucer.AI includes built-in providers for Anthropic, OpenAI, and Gemini. Providers are registered with `AddSaucerAI()` and configured with the standard .NET options pattern.

## Registering Providers

Each provider gets a name, which is how orchestrators reference it.

```csharp
builder.Services.AddSaucerAI()
    .WithAnthropic("Anthropic", options =>
    {
        options.ApiKey = builder.Configuration["Anthropic:ApiKey"];
        options.DefaultModel = "claude-sonnet-4-20250514";
        options.DisplayName = "Claude Sonnet";
    })
    .WithOpenAI("OpenAI", options =>
    {
        options.ApiKey = builder.Configuration["OpenAI:ApiKey"];
        options.DefaultModel = "gpt-4o";
        options.DisplayName = "GPT-4o";
    })
    .WithGemini("Gemini", options =>
    {
        options.ApiKey = builder.Configuration["Gemini:ApiKey"];
        options.DefaultModel = "gemini-2.5-flash";
        options.DisplayName = "Gemini Flash";
    });
```

## Provider Options

All providers share a common set of options through `LlmProviderOptions`:

| Option | Description |
|---|---|
| `ApiKey` | API key for the provider |
| `ApiUrl` | Base URL override (useful for proxies or custom endpoints) |
| `DefaultModel` | Model identifier used when no override is specified |
| `DisplayName` | Human-readable name for UI display |
| `Description` | Optional description |
| `TimeoutSeconds` | Request timeout |
| `Tags` | Arbitrary key-value metadata |
| `ResilienceOptions` | HTTP resilience policy configuration |

Anthropic also supports `ApiVersion` for specifying the API version header.

## Multiple Configurations per Provider

You can register the same provider type multiple times with different names and models:

```csharp
builder.Services.AddSaucerAI()
    .WithOpenAI("gpt4-fast", options =>
    {
        options.ApiKey = builder.Configuration["OpenAI:ApiKey"];
        options.DefaultModel = "gpt-4o-mini";
        options.DisplayName = "GPT-4o Mini (Fast)";
    })
    .WithOpenAI("gpt4-reasoning", options =>
    {
        options.ApiKey = builder.Configuration["OpenAI:ApiKey"];
        options.DefaultModel = "gpt-4o";
        options.DisplayName = "GPT-4o (Reasoning)";
    });
```

Orchestrators can then target specific configurations by name.

## HTTP Resilience

Providers support `Microsoft.Extensions.Http.Resilience` policies for retry, circuit breaker, and timeout handling:

```csharp
.WithAnthropic("Anthropic", options =>
{
    options.ApiKey = builder.Configuration["Anthropic:ApiKey"];
    options.DefaultModel = "claude-sonnet-4-20250514";
    options.ResilienceOptions = new HttpStandardResilienceOptions
    {
        AttemptTimeout = new HttpTimeoutStrategyOptions
        {
            Timeout = TimeSpan.FromSeconds(60)
        },
        TotalRequestTimeout = new HttpTimeoutStrategyOptions
        {
            Timeout = TimeSpan.FromSeconds(125)
        },
        Retry = new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 1
        },
        CircuitBreaker = new HttpCircuitBreakerStrategyOptions
        {
            SamplingDuration = TimeSpan.FromSeconds(120)
        }
    };
})
```

## Per-Request Provider Override

Orchestrators have a default LLM, but you can override it per-request using `OrchestrationPolicy`:

```csharp
var result = await orchestrator.OrchestrateAsync(
    sessionId: sessionId,
    userMessage: message,
    policy: new OrchestrationPolicy
    {
        LlmProvider = "gpt4-reasoning"  // Override for this request
    });
```

This works because the context window uses provider-agnostic `FrameworkMessage` types. The conversation history doesn't need to change when you switch providers.
