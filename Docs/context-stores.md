# Context Stores

A context store manages conversation state for an orchestration service. It controls how session context is created, retrieved, persisted, and cleaned up.

## Default In-Memory Store

The simplest setup uses the built-in `InMemoryContextStore`:

```csharp
builder.Services.AddOrchestrationService("Assistant")
    .WithSystemPrompt("You are a helpful assistant.")
    .WithDefaultLlm("Anthropic")
    .WithContextStore();  // In-memory, lost on restart
```

This works well for stateless or ephemeral interactions like single-turn queries.

## The Interface

```csharp
public interface ISessionContextStore
{
    Task<SessionContext> GetContextAsync(string orchestratorRole, string sessionId);
    Task<SessionContext> GetOrCreateContextAsync(string orchestratorRole, string sessionId);
    Task ContextChangedAsync(string orchestratorRole, string sessionId,
        SessionContext context, ContextChangeSource changeSource);
    Task SessionInactiveAsync(string orchestratorRole, string sessionId, DateTime lastAccessed);
    Task<IEnumerable<string>> GetSessionIdsAsync(string orchestratorRole);
    Task<bool> DeleteAsync(string orchestratorRole, string sessionId);
}
```

## Implementing a Custom Store

Implement `ISessionContextStore` to persist conversations to a database, cache, or any other storage:

```csharp
public class DatabaseContextStore : ISessionContextStore
{
    private readonly IDatabase _db;

    public DatabaseContextStore(IDatabase db)
    {
        _db = db;
    }

    public async Task<SessionContext> GetOrCreateContextAsync(
        string orchestratorRole, string sessionId)
    {
        // Load persisted state, or create a new context
        var state = await _db.FindAsync(orchestratorRole, sessionId);
        if (state != null)
        {
            var context = new SessionContext(sessionId);
            context.RestoreState(state);
            return context;
        }
        return new SessionContext(sessionId);
    }

    public async Task ContextChangedAsync(
        string orchestratorRole, string sessionId,
        SessionContext context, ContextChangeSource changeSource)
    {
        // Called after every orchestration turn
        // ExportState() gives you a serializable snapshot
        var state = context.ExportState();
        await _db.UpsertAsync(orchestratorRole, sessionId, state);
    }

    public async Task<SessionContext> GetContextAsync(
        string orchestratorRole, string sessionId)
    {
        // Return null if not found — used for optional lookups
        var state = await _db.FindAsync(orchestratorRole, sessionId);
        if (state == null) return null;

        var context = new SessionContext(sessionId);
        context.RestoreState(state);
        return context;
    }

    public async Task SessionInactiveAsync(
        string orchestratorRole, string sessionId, DateTime lastAccessed)
    {
        // Called when a session becomes inactive — opportunity to archive or clean up
    }

    public async Task<IEnumerable<string>> GetSessionIdsAsync(string orchestratorRole)
    {
        return await _db.GetSessionIdsAsync(orchestratorRole);
    }

    public async Task<bool> DeleteAsync(string orchestratorRole, string sessionId)
    {
        return await _db.DeleteAsync(orchestratorRole, sessionId);
    }
}
```

## Registering a Custom Store

```csharp
builder.Services.AddOrchestrationService("Customer-Chat")
    .WithSystemPromptFile("Prompts/CustomerChat.md")
    .WithDefaultLlm("Anthropic")
    .WithContextStore<DatabaseContextStore>()
    .WithAgentTools();
```

Or with a factory:

```csharp
.WithContextStore(sp =>
{
    var db = sp.GetRequiredService<IDatabase>();
    return new DatabaseContextStore(db);
})
```

## State Export and Import

`SessionContext` supports exporting and restoring its state through `SessionContextState`. This is a serializable snapshot of the full conversation — messages, metadata, and timestamps — suitable for storage in any format (JSON, binary, etc.).

Each orchestration service has its own context store instance. Contexts are scoped by `orchestratorRole` (the service name) and `sessionId`, so multiple orchestrators can manage independent conversation histories for the same user.

## Direct Context Access

Outside of orchestration, you can access context safely through the orchestration service:

```csharp
// Read context state
var state = await orchestrator.GetContextStateAsync(sessionId);

// Modify context with locking
await orchestrator.WithLockedContextAsync(sessionId, async context =>
{
    // Safe to read/modify context here
    return ContextChangeResult.Changed;
});

// Delete a session
await orchestrator.DeleteSessionAsync(sessionId);
```
