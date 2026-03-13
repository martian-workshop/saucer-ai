# Saucer.AI

**A highly composable AI orchestration library for C# (.NET) with native ASP pipeline support.**

[![License: Proprietary](https://img.shields.io/badge/License-Proprietary-blue.svg)](LICENSE)
[![.NET 8+](https://img.shields.io/badge/.NET-8.0+-512BD4)](https://dotnet.microsoft.com/)

## What is Saucer.AI?

Saucer.AI is an orchestration library used internally by The Martian Workshop for building AI-powered .NET applications. Using the framework, AI can be readily added to any .NET application with a few lines of code. This puts AI in easy reach of any application whether it runs on a server or the desktop without deep dependencies or vendor lock-in. 

Saucer.AI provides multi-provider LLM support, context management, an attribute-based tool system, and named orchestration services that integrate directly into Microsoft's ServiceProvider pipeline.

## Architecture and Design

Rather than bury core concepts in unnecessary abstractions, Saucer.AI assumes a basic understanding of what an LLM orchestration loop does — send messages, get responses, execute tool calls. This approach provides clarity in the framework and transparent, composable building blocks.

The result is that each agent runtime registered in the DI configuration is self-documenting. Extensibility becomes natural and readily composable. There are no hidden switches or abstracted convenience wrappers that you'll need to reverse-engineer later when you need to specialize your runtime(s).

This also means AI coding assistants will immediately understand your architecture just by reading the startup code. For example, you can paste this code snippet into any LLM — even with no knowledge of Saucer.AI or the [Criterion Companion](https://github.com/martian-workshop/Criterion) demo application — and it will immediately be able to tell you what it does and what each component is for:

```csharp
services.AddOrchestrationService("Criterion-Chat")
    .WithSystemPromptFile("Prompts/Cineguide.md")
    .WithDefaultLlm("Anthropic")
    .WithContextStore<CriterionContextStore>()
    .WithObserver<CriterionOrchestrationObserver>()
    .WithAgentTools();
```

This value cannot be understated.

## Key Features

- **Multi-provider LLM support** — Built-in providers for Anthropic, OpenAI, and Gemini with a provider-agnostic framework layer. By abstracting providers and the underlying context, applications can switch between providers readily with a single configuration change, or even mid-conversation or orchestration turn. Tweak context and choose providers based on what an LLM is good at, business choices, or even ad hoc based on availability (429) without recycling the current context window. 
- **Named orchestration services** — Register multiple independent orchestrators, each with their own system prompt, default LLM, context store, observer, and tool set. Inject by name using .NET keyed services.
- **Native AgentTool support** — Define tools with `[AgentTool]` and `[ToolParam]` attributes. Assembly scanning discovers and registers them automatically. Tools execute locally in-process so local tools are instantly executable without moving them to external MCP servers. (If your code can call it, so can the LLM.) 
- **Observable orchestration** — Typed event pipeline with `IOrchestrationObserver` for reacting to orchestration lifecycle events: before LLM calls, tool approvals, tool execution, completion, and errors. Observers have access to everything the orchestrator sees. 
- **Session context management** — Pluggable `ISessionContextStore` with state export and import for persisting conversations across requests. Default in-memory store included; bring your own for database-backed persistence. 
- **Token tracking** — Prompt and completion token usage surfaced on every orchestration result.
- **Thread-safe orchestrations** — Saucer.AI supports threadsafe context/orchestration management out of the box. Built-in session locking ensures safe concurrent access from multiple orchestration requests.
- **HTTP resilience** — Retry, circuit breaker, and timeout policies configured as first-class provider options via `Microsoft.Extensions.Http.Resilience`.
- **Composable toolsets** — Assign all registered tools or specific tool service classes to individual orchestrators.
- **Streaming support** — `IAsyncEnumerable<OrchestrationEvent>` streaming for real-time UI updates.
- **Native DI integration** — Built on `IServiceCollection` and keyed services. No wrapper abstractions — just standard ASP.NET Core patterns.

## System Requirements

- **.NET** 8.0 or later
- **OS Support**: Windows, macOS, Linux

## Installation

*Saucer.AI is an internal framework used by The Martian Workshop and its partners. Source code is not currently distributed.*

## Quickstart

### 1. Register Providers and Create an Orchestrator

The simplest setup: one LLM provider, one orchestrator, send a message.

```csharp
using Saucer.AI.Extensions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);

// Register an LLM provider
builder.Services.AddSaucerAI()
    .WithAnthropic("Anthropic", options =>
    {
        options.ApiKey = builder.Configuration["Anthropic:ApiKey"];
        options.DefaultModel = "claude-sonnet-4-20250514";
    });

// Create an orchestration service with a system prompt
builder.Services.AddOrchestrationService()
    .WithSystemPrompt("You are a helpful assistant.")
    .WithDefaultLlm("Anthropic")
    .WithContextStore();   // In-memory conversation context

var app = builder.Build();

// Use the orchestrator in an endpoint
app.MapPost("/chat", async (
    IOrchestrationService orchestrator,
    ChatRequest request) =>
{
    // OrchestrateAsync runs the full orchestration loop:
    // send the message, get the LLM response, execute any tool calls, repeat until done
    var result = await orchestrator.OrchestrateAsync(
        sessionId: request.SessionId,
        userMessage: request.Message,
        policy: null);  // null = use defaults configured above

    return new
    {
        result.FinalMessage,
        result.Model,
        result.TotalPromptTokens,
        result.TotalCompletionTokens,
        result.TotalToolCalls
    };
});

app.Run();
```

### 2. Named Orchestrators with Tools

Real applications need multiple agents with different capabilities. Saucer.AI makes this natural with named services and attribute-based tools.

#### Define tools

Tools are regular C# classes. Saucer.AI discovers them by attribute and handles schema generation, parameter binding, and invocation automatically. Tools execute locally in your process.

```csharp
using Saucer.AI.Tools.Attributes;

[AgentToolService("Weather Tools")]
public class WeatherTools
{
    private readonly IWeatherApi _weatherApi;

    public WeatherTools(IWeatherApi weatherApi)
    {
        _weatherApi = weatherApi;  // Constructor injection works as expected
    }

    [AgentTool(
        Name = "get_forecast",
        Description = "Gets the weather forecast for a city.")]
    public async Task<Forecast> GetForecastAsync(
        [ToolParam(Description = "City name (e.g., 'Seattle', 'London')")] string city,
        [ToolParam(Description = "Number of days (1-7)", Required = false)] int days = 3)
    {
        return await _weatherApi.GetForecastAsync(city, days);
    }

    [AgentTool(
        Name = "get_alerts",
        Description = "Gets active weather alerts for a region.")]
    public async Task<IEnumerable<Alert>> GetAlertsAsync(
        [ToolParam(Description = "Region or state code (e.g., 'WA', 'CA')")] string region)
    {
        return await _weatherApi.GetActiveAlertsAsync(region);
    }
}

[AgentToolService("Calendar Tools")]
public class CalendarTools
{
    [AgentTool(
        Name = "get_events",
        Description = "Gets upcoming calendar events for a user.")]
    public async Task<IEnumerable<CalendarEvent>> GetEventsAsync(
        [ToolParam(Description = "The user ID")] string userId,
        [ToolParam(Description = "Number of days ahead to look", Required = false)] int days = 7)
    {
        // Your calendar logic here
    }
}
```

#### Register named orchestrators

Each orchestrator gets its own identity, system prompt, LLM, and tool set.

```csharp
using Saucer.AI.Extensions.DependencyInjection;
using Saucer.AI.Extensions.DependencyInjection.Tools;

// Scan the assembly for [AgentToolService] classes and register them
builder.Services.AddAgentTools();

// Register LLM providers
builder.Services.AddSaucerAI()
    .WithAnthropic("Anthropic", options =>
    {
        options.ApiKey = builder.Configuration["Anthropic:ApiKey"];
        options.DefaultModel = "claude-sonnet-4-20250514";
    })
    .WithGemini("Gemini", options =>
    {
        options.ApiKey = builder.Configuration["Gemini:ApiKey"];
        options.DefaultModel = "gemini-2.5-flash";
    });

// A conversational assistant with access to ALL registered tools
builder.Services.AddOrchestrationService("Assistant")
    .WithSystemPromptFile("Prompts/Assistant.md")
    .WithDefaultLlm("Anthropic")
    .WithContextStore()
    .WithAgentTools();  // All [AgentToolService] classes

// A fast, stateless query agent with only weather tools
builder.Services.AddOrchestrationService("Weather")
    .WithSystemPrompt("You are a concise weather information service. Answer with data, not commentary.")
    .WithDefaultLlm("Gemini")
    .WithContextStore()
    .WithAgentToolset<WeatherTools>();  // Only WeatherTools
```

#### Inject by name

```csharp
app.MapPost("/chat", async (
    [FromKeyedServices("Assistant")] IOrchestrationService assistant,
    ChatRequest request) =>
{
    var result = await assistant.OrchestrateAsync(request.SessionId, request.Message, null);
    return result.FinalMessage;
});

app.MapGet("/weather/{city}", async (
    [FromKeyedServices("Weather")] IOrchestrationService weather,
    string city) =>
{
    var result = await weather.OrchestrateAsync(
        sessionId: Guid.NewGuid().ToString(),
        userMessage: $"What's the forecast for {city}?",
        policy: null);
    return result.FinalMessage;
});
```

### 3. Streaming with Observers

For real-time UI updates, use `OrchestrateStreamAsync` to receive typed orchestration events as they happen. For cross-cutting concerns like logging or analytics, register an `IOrchestrationObserver`.

#### Stream orchestration events

```csharp
app.MapPost("/chat/stream", async (
    [FromKeyedServices("Assistant")] IOrchestrationService assistant,
    ChatRequest request,
    HttpContext httpContext) =>
{
    httpContext.Response.ContentType = "text/event-stream";

    await foreach (var evt in assistant.OrchestrateStreamAsync(
        request.SessionId, request.Message, policy: null))
    {
        // Each event is a typed OrchestrationEvent subclass:
        //   OrchestrationStartedEvent   — orchestration loop began
        //   LlmResponseGeneratedEvent   — LLM returned a response
        //   ToolCallRequestedEvent      — LLM wants to call a tool
        //   ToolExecutedEvent           — tool finished executing
        //   OrchestrationCompletedEvent — final result ready
        //   OrchestrationErrorEvent     — an error occurred

        var json = JsonSerializer.Serialize(evt);
        await httpContext.Response.WriteAsync($"data: {json}\n\n");
        await httpContext.Response.Body.FlushAsync();
    }
});
```

#### Register an observer

Observers participate in the orchestration lifecycle. They can log events, enforce policies, or approve tool calls before execution.

```csharp
using Saucer.AI.Orchestrator.Events;
using Saucer.AI.Orchestrator.Models;

public class AuditObserver : OrchestrationObserver
{
    private readonly ILogger<AuditObserver> _logger;

    public AuditObserver(ILogger<AuditObserver> logger)
    {
        _logger = logger;
    }

    public override Task OnOrchestrationStartingAsync(OrchestrationContext context)
    {
        _logger.LogInformation("Orchestration started: {Id}", context.OrchestrationId);
        return Task.CompletedTask;
    }

    // Approve or reject tool calls before they execute
    public override Task<ToolApprovalResult> OnToolCallRequestedAsync(
        ToolCallRequest request, OrchestrationContext context)
    {
        _logger.LogInformation("Tool requested: {Tool}", request.ToolName);
        return Task.FromResult(ToolApprovalResult.Approved);
    }

    public override Task OnToolExecutedAsync(
        ToolCallResult result, OrchestrationContext context)
    {
        _logger.LogInformation("Tool executed: {Tool} in {Duration}ms",
            result.ToolName, result.ExecutionDuration.TotalMilliseconds);
        return Task.CompletedTask;
    }

    public override Task OnOrchestrationCompleteAsync(
        Orchestrator orchestrator, OrchestratorResult result, OrchestrationContext context)
    {
        _logger.LogInformation(
            "Completed: {Tokens} tokens, {Tools} tool calls, {Iterations} iterations",
            result.TotalPromptTokens + result.TotalCompletionTokens,
            result.TotalToolCalls,
            result.TotalIterations);
        return Task.CompletedTask;
    }
}
```

#### Wire it up

```csharp
builder.Services.AddOrchestrationService("Assistant")
    .WithSystemPromptFile("Prompts/Assistant.md")
    .WithDefaultLlm("Anthropic")
    .WithContextStore<MyDatabaseContextStore>()
    .WithObserver<AuditObserver>()
    .WithAgentTools();
```

## Where to Go Next

- [Tool Authoring Guide](docs/tools.md) — Creating and organizing agent tool services
- [Provider Configuration](docs/providers.md) — Setting up LLM providers with resilience policies
- [Orchestration Services](docs/orchestration.md) — Named services, policies, and LLM options
- [Observers](docs/observers.md) — Lifecycle events, tool approval, and custom observers
- [Context Stores](docs/context-stores.md) — Session persistence and custom store implementations
- [Architecture Overview](docs/architecture.md) — Design principles and component relationships

## About

Saucer.AI is built and maintained by [The Martian Workshop](https://www.martianworkshop.com).

## License

Saucer.AI is proprietary software developed by [The Martian Workshop](https://www.martianworkshop.com). All rights reserved.

Licensing is available for partners and collaborators. For inquiries, please reach out through [martianworkshop.com](https://www.martianworkshop.com).
