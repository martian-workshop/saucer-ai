# Architecture Overview

Saucer.AI is organized around a small number of well-defined interfaces. Rather than deep class hierarchies, components are composed through configuration — each orchestration service is assembled from a provider, a context store, an observer, and a tool set.

## Core Components

```mermaid
classDiagram
    direction TB

    IOrchestrationService *-- Orchestrator
    IOrchestrationService *-- ISessionContextStore
    IOrchestrationService *-- IOrchestrationObserver
    IOrchestrationService *-- ILlmProviderRegistry

    Orchestrator --> ILlmProvider : sends requests
    Orchestrator --> IAgentTools : executes tools
    Orchestrator --> SessionContext : reads/writes

    class IOrchestrationService {
        <<Interface>>
        +String SystemPrompt
        +String OrchestratorRole
        +String DefaultProviderName
        +ILlmProviderRegistry ProviderRegistry
        +OrchestrateAsync(sessionId, message, policy) Task~OrchestratorResult~
        +OrchestrateStreamAsync(sessionId, message, policy) IAsyncEnumerable~OrchestrationEvent~
        +WithLockedContextAsync(sessionId, operation) Task
        +DeleteSessionAsync(sessionId) Task~bool~
    }

    class Orchestrator {
        Runs the core orchestration loop
        Send → Receive → Tool Call → Repeat
        Tracks tokens, iterations, tool calls
    }

    class OrchestratorResult {
        +String FinalMessage
        +String Model
        +int TotalPromptTokens
        +int TotalCompletionTokens
        +int TotalIterations
        +int TotalToolCalls
        +bool HasError
        +bool WasCancelled
    }

    Orchestrator ..> OrchestratorResult : produces
```

## Provider Abstraction

All LLM providers implement `ILlmProvider`. The framework layer defines provider-agnostic message types — `FrameworkRequest`, `FrameworkResponse`, `FrameworkMessage`, and `ContentPart`. Provider-specific details never leak beyond the adapter boundary.

This is what enables mid-conversation provider switching. The context window holds `FrameworkMessage` objects, not provider-specific types.

```mermaid
classDiagram
    direction LR

    ILlmProvider <|.. AnthropicProviderAdapter
    ILlmProvider <|.. OpenAIProviderAdapter
    ILlmProvider <|.. GeminiProviderAdapter

    AnthropicProviderAdapter --> AnthropicProvider : HTTP client
    OpenAIProviderAdapter --> OpenAIProvider : HTTP client
    GeminiProviderAdapter --> GeminiProvider : HTTP client

    class ILlmProvider {
        <<Interface>>
        +SendAsync(FrameworkRequest) Task~FrameworkResponse~
    }

    class FrameworkRequest {
        +List~FrameworkMessage~ Messages
        +String SystemPrompt
        +ModelParameters Parameters
        +List~McpToolDefinition~ Tools
    }

    class FrameworkResponse {
        +FrameworkMessage Message
        +TokenUsage Usage
        +StopReason StopReason
        +String Model
    }

    class FrameworkMessage {
        +String Role
        +List~ContentPart~ Content
    }

    class ContentPart {
        <<Abstract>>
    }
    ContentPart <|-- TextPart
    ContentPart <|-- ToolCallPart
    ContentPart <|-- ToolResultPart
    ContentPart <|-- DataPart

    FrameworkRequest *-- FrameworkMessage
    FrameworkResponse *-- FrameworkMessage
    FrameworkMessage *-- ContentPart

    ILlmProvider ..> FrameworkRequest : accepts
    ILlmProvider ..> FrameworkResponse : returns
```

## Observer Pipeline

Observers participate in every phase of the orchestration lifecycle. The base `OrchestrationObserver` class provides virtual no-op methods so implementations only need to override what they care about.

```mermaid
classDiagram

    IOrchestrationObserver <|.. OrchestrationObserver

    class IOrchestrationObserver {
        <<Interface>>
        +OnOrchestrationStartingAsync(context)
        +OnBeforeLlmCallAsync(context)
        +OnResponseGeneratedAsync(response, context)
        +OnToolCallRequestedAsync(request, context) Task~ToolApprovalResult~
        +OnToolExecutedAsync(result, context)
        +OnOrchestrationCompleteAsync(orchestrator, result, context)
    }

    class OrchestrationObserver {
        <<Base Class>>
        All methods are virtual no-ops
        Override only what you need
    }

    class OrchestrationContext {
        +String OrchestrationId
        +ILlmProvider Provider
        +LlmOptions LlmOptions
        +int TotalPromptTokens
        +int TotalCompletionTokens
        +int TotalToolCalls
        +int IterationCnt
        +Dictionary RequestContext
        +Dictionary RequestData
    }

    IOrchestrationObserver ..> OrchestrationContext : receives
    IOrchestrationObserver ..> ToolCallRequest : approves/rejects
    IOrchestrationObserver ..> ToolCallResult : observes

    class ToolCallRequest {
        +String Id
        +String ToolName
        +JsonNode Parameters
        +int IterationNumber
    }

    class ToolCallResult {
        +String ToolCallId
        +String ToolName
        +String Content
        +bool IsError
        +TimeSpan ExecutionDuration
    }
```

## Agent Tools

Tools are regular C# classes discovered by attribute scanning. The framework handles schema generation, parameter binding, and invocation. Tools execute locally in-process.

```mermaid
classDiagram
    direction TB

    AgentToolScanner ..> AgentToolRegistry : discovers and populates
    AgentToolRegistry ..|> IAgentTools
    AgentToolRegistry *-- ToolMetadata

    class IAgentTools {
        <<Interface>>
    }

    class AgentToolScanner {
        Scans assemblies for
        [AgentToolService] classes
    }

    class AgentToolRegistry {
        Stores tool definitions
        Invokes tools by name
    }

    class ToolMetadata {
        +String Name
        +String Description
        +ParameterMetadata[] Parameters
    }

    class AgentToolServiceAttribute {
        <<Attribute>>
        Marks a class as a tool service
    }

    class AgentToolAttribute {
        <<Attribute>>
        +String Name
        +String Description
        Marks a method as a callable tool
    }

    class ToolParamAttribute {
        <<Attribute>>
        +String Description
        +bool Required
        Describes a tool parameter
    }

    AgentToolServiceAttribute --> AgentToolScanner : discovered by
    AgentToolAttribute --> ToolMetadata : generates
    ToolParamAttribute --> ToolMetadata : generates
```

## Design Principles

- **Composability over inheritance** — Features are assembled through builder configuration, not deep class hierarchies.
- **Transparency over convenience** — The DI configuration describes the full agent architecture. Nothing is hidden behind defaults that are hard to discover.
- **Standard .NET patterns** — `IServiceCollection`, keyed services, options pattern, `IAsyncEnumerable`. If you know ASP.NET Core, you already know how to use Saucer.AI.
- **Provider agnosticism at the core** — Business logic never touches provider-specific types. Switching providers is a configuration change, not a code change.
