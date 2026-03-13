# Observers

Observers hook into the orchestration lifecycle. They see every LLM call, every tool request, and every result. Use them for logging, analytics, tool approval, or custom business logic.

## The Interface

`IOrchestrationObserver` defines six lifecycle methods:

```csharp
public interface IOrchestrationObserver
{
    Task OnOrchestrationStartingAsync(OrchestrationContext context);
    Task OnBeforeLlmCallAsync(OrchestrationContext context);
    Task OnResponseGeneratedAsync(FrameworkResponse response, OrchestrationContext context);
    Task<ToolApprovalResult> OnToolCallRequestedAsync(ToolCallRequest request, OrchestrationContext context);
    Task OnToolExecutedAsync(ToolCallResult result, OrchestrationContext context);
    Task OnOrchestrationCompleteAsync(Orchestrator orchestrator, OrchestratorResult result, OrchestrationContext context);
}
```

## The Base Class

`OrchestrationObserver` provides virtual no-op implementations of every method. Extend it and override only what you need:

```csharp
public class LoggingObserver : OrchestrationObserver
{
    private readonly ILogger<LoggingObserver> _logger;

    public LoggingObserver(ILogger<LoggingObserver> logger)
    {
        _logger = logger;
    }

    public override Task OnOrchestrationStartingAsync(OrchestrationContext context)
    {
        _logger.LogInformation("Orchestration {Id} started", context.OrchestrationId);
        return Task.CompletedTask;
    }

    public override Task OnOrchestrationCompleteAsync(
        Orchestrator orchestrator, OrchestratorResult result, OrchestrationContext context)
    {
        _logger.LogInformation(
            "Orchestration {Id} completed — {Tokens} tokens, {Tools} tool calls",
            context.OrchestrationId,
            result.TotalPromptTokens + result.TotalCompletionTokens,
            result.TotalToolCalls);
        return Task.CompletedTask;
    }
}
```

## Tool Approval

`OnToolCallRequestedAsync` is called before every tool execution. Return `ToolApprovalResult.Approved` to allow it, or reject it to prevent execution:

```csharp
public override Task<ToolApprovalResult> OnToolCallRequestedAsync(
    ToolCallRequest request, OrchestrationContext context)
{
    // Example: block destructive operations in read-only mode
    if (request.ToolName.StartsWith("delete_"))
    {
        return Task.FromResult(ToolApprovalResult.Rejected("Delete operations are not allowed."));
    }

    return Task.FromResult(ToolApprovalResult.Approved);
}
```

When a tool call is rejected, the rejection reason is sent back to the LLM as the tool result, allowing it to adjust its approach.

## OrchestrationContext

Every observer method receives `OrchestrationContext`, which provides access to the current state of the orchestration:

| Property | Description |
|---|---|
| `OrchestrationId` | Unique ID for this orchestration run |
| `Provider` | The active `ILlmProvider` |
| `LlmOptions` | Current LLM options |
| `TotalPromptTokens` | Running prompt token count |
| `TotalCompletionTokens` | Running completion token count |
| `TotalToolCalls` | Running tool call count |
| `IterationCnt` | Current iteration in the orchestration loop |
| `RequestContext` | String dictionary from `OrchestrationPolicy` — useful for passing user IDs, locale, or other request-scoped data |
| `RequestData` | Object dictionary for passing complex data |

## Registering an Observer

```csharp
builder.Services.AddOrchestrationService("Assistant")
    .WithSystemPromptFile("Prompts/Assistant.md")
    .WithDefaultLlm("Anthropic")
    .WithContextStore()
    .WithObserver<LoggingObserver>()
    .WithAgentTools();
```

Or with a factory for more control:

```csharp
.WithObserver(sp =>
{
    var logger = sp.GetRequiredService<ILogger<LoggingObserver>>();
    return new LoggingObserver(logger);
})
```

## Streaming Events

When using `OrchestrateStreamAsync`, the orchestration lifecycle is surfaced as typed `OrchestrationEvent` subclasses:

| Event | When |
|---|---|
| `OrchestrationStartedEvent` | Orchestration loop begins |
| `LlmResponseGeneratedEvent` | LLM returns a response |
| `ToolCallRequestedEvent` | LLM wants to call a tool |
| `ToolExecutedEvent` | Tool finished executing |
| `OrchestrationCompletedEvent` | Final result ready |
| `OrchestrationErrorEvent` | An error occurred |

Observers and streaming events work independently — observers handle cross-cutting concerns at the service level, while streaming events deliver real-time updates to the caller.
