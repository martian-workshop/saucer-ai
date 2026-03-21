# Tool Authoring Guide

Saucer.AI's tool system lets you expose regular C# methods to the LLM. Tools execute locally in your process — if your code can call it, so can the LLM.

## Defining a Tool Service

Mark a class with `[AgentToolService]` and its methods with `[AgentTool]`:

```csharp
using Saucer.AI.Tools.Attributes;

[AgentToolService("Inventory Tools")]
public class InventoryTools
{
    private readonly IInventoryService _inventory;

    // Constructor injection works as expected
    public InventoryTools(IInventoryService inventory)
    {
        _inventory = inventory;
    }

    [AgentTool(
        Name = "check_stock",
        Description = "Checks the current stock level for a product.")]
    public async Task<StockInfo> CheckStockAsync(
        [ToolParam(Description = "The product SKU")] string sku)
    {
        return await _inventory.GetStockAsync(sku);
    }

    [AgentTool(
        Name = "search_products",
        Description = "Searches the product catalog by keyword.")]
    public async Task<IEnumerable<Product>> SearchProductsAsync(
        [ToolParam(Description = "Search keyword", Required = true)] string query,
        [ToolParam(Description = "Maximum results to return", Required = false)] int limit = 10)
    {
        return await _inventory.SearchAsync(query, limit);
    }
}
```

That's it. Saucer.AI handles schema generation, parameter binding, and invocation.

## The Attributes

### `[AgentToolService("name")]`

Marks a class as a tool service. The name is used for grouping and identification.

- The class can have constructor dependencies — they're resolved from the DI container.
- One class can contain multiple `[AgentTool]` methods.
- The class is the unit of composition when using `WithAgentToolset<T>()`.

### `[AgentTool]`

Marks a method as a callable tool.

| Property | Description |
|---|---|
| `Name` | The tool name the LLM will use (e.g., `"check_stock"`) |
| `Description` | What the tool does — this is what the LLM reads to decide when to call it |

### `[ToolParam]`

Describes a method parameter.

| Property | Description |
|---|---|
| `Description` | What the parameter is for — helps the LLM provide correct values |
| `Required` | Whether the parameter is required (default: `true`) |

## Registering Tools

### Scan the entry assembly

The simplest approach — discovers all `[AgentToolService]` classes in your application:

```csharp
builder.Services.AddAgentTools()
    .FromEntryAssembly();
```

### Give all tools to an orchestrator

```csharp
builder.Services.AddOrchestrationService("Assistant")
    .WithSystemPrompt("You are a helpful assistant.")
    .WithDefaultLlm("Anthropic")
    .WithContextStore()
    .WithAgentTools();  // All registered tool services
```

### Give specific tools to an orchestrator

```csharp
builder.Services.AddOrchestrationService("Inventory-Agent")
    .WithSystemPrompt("You help with inventory queries.")
    .WithDefaultLlm("Anthropic")
    .WithContextStore()
    .WithAgentToolset<InventoryTools>();  // Only InventoryTools
```

You can chain multiple `WithAgentToolset<T>()` calls to compose a custom tool set:

```csharp
builder.Services.AddOrchestrationService("Support-Agent")
    .WithSystemPrompt("You handle customer support requests.")
    .WithDefaultLlm("Anthropic")
    .WithContextStore()
    .WithAgentToolset<InventoryTools>()
    .WithAgentToolset<OrderTools>()
    .WithAgentToolset<CustomerTools>();
```

## How It Works

1. **Startup** — `AddAgentTools()` scans for `[AgentToolService]` classes. For each `[AgentTool]` method, it generates MCP-compatible JSON schemas from the method signature and `[ToolParam]` attributes.

2. **Orchestration** — When an orchestrator sends a request to the LLM, the tool schemas are included. The LLM sees the tool names, descriptions, and parameter schemas.

3. **Tool call** — When the LLM responds with a tool call, the orchestrator matches it to the registered tool, deserializes the parameters, and invokes the method.

4. **Result** — The method's return value is serialized and sent back to the LLM as a tool result. The orchestration loop continues.

Tools are just methods. They can do anything your application code can do — query a database, call an API, read a file, compute a value. No external services or protocol adapters required.
