# chuks_agent

Structured **agent runtime** for Chuks. Production-grade AI agents with tool calling, structured concurrency, multi-agent coordination, tagged memory, streaming, retries, per-step deadlines, per-model cost accounting, and guaranteed cancellation.

The killer feature: `task.timeout(60000)` cancels the **entire agent tree** — every sub-agent, every LLM call, every in-flight tool execution. Guaranteed. No runaway LLM costs.

---

## Install

```json
{
  "dependencies": {
    "@chuks/ai": "1.0.0"
  }
}
```

```bash
chuks add @chuks/agent
```

---

## Quick Start

```chuks
import { agent, tool, tools } from "pkg/@chuks/agent"
import { ai } from "pkg/@chuks/ai"

var searchTool = tools.webSearch({ "provider": "brave", "apiKey": "..." })
var readTool   = tools.readURL({ "maxLength": 8000 })

var researcher = agent.create({
    "name":         "researcher",
    "model":        ai.openai("gpt-4o"),
    "tools":        [searchTool, readTool],
    "systemPrompt": "You are a research assistant. Find accurate information.",
    "maxSteps":     10,
})

var result = await researcher.run("What are the latest features in Chuks?", null)
println(result.answer)
println("Steps: "  + string(result.totalSteps))
println("Tokens: " + string(result.tokens))
println("Cost: $"  + string(result.cost))
```

---

## What an Agent Is

```
Normal LLM call:
  prompt → LLM → answer (one shot)

Agent:
  prompt → LLM → "I need to search" → calls search tool
         → gets results → LLM → "I need to read that URL"
         → calls readURL → gets content → LLM → "Now I know"
         → final answer
```

The LLM is the brain. Tools are the hands. The loop runs until the model has enough information (or a guardrail — `maxSteps`, `timeout`, `maxCost` — fires).

---

## Exported Symbols

| Symbol                                                    | Kind                     | Purpose                                                                 |
| --------------------------------------------------------- | ------------------------ | ----------------------------------------------------------------------- |
| `agent`                                                   | singleton `AgentModule`  | `agent.create({...})` builds an `Agent`.                                |
| `tool`                                                    | singleton `ToolModule`   | `tool.create({...})` and `tool.fromAgent(a)`.                           |
| `tools`                                                   | singleton `BuiltinTools` | Built-in tool factories — `tools.webSearch(...)`, etc.                  |
| `crew`                                                    | singleton `CrewModule`   | `crew.sequential([...])`, `crew.parallel([...])`.                       |
| `Agent`                                                   | class                    | The agent runtime (created via `agent.create`).                         |
| `AgentTool`                                               | class                    | A tool with `safeExecute`, `execute`, `toSchema`.                       |
| `AgentEvent`                                              | class                    | One event emitted by `Agent.stream` over a typed channel.               |
| `SequentialCrew`                                          | class                    | The pipeline type returned by `crew.sequential([...])`.                 |
| `MemoryEntry`                                             | class                    | A single tagged memory entry.                                           |
| `AgentModule`, `ToolModule`, `BuiltinTools`, `CrewModule` | classes                  | The facade classes (singletons exported above; rarely needed directly). |
| `Message`                                                 | `dataType`               | One chat-history entry.                                                 |
| `ToolCall`                                                | `dataType`               | An LLM tool-invocation request.                                         |
| `ToolResult`                                              | `dataType`               | Result envelope from `AgentTool.safeExecute`.                           |
| `AgentStep`                                               | `dataType`               | One iteration of the agent loop.                                        |
| `AgentRunResult`                                          | `dataType`               | Full return shape of `Agent.run` / `crew.*.run`.                        |
| `CrewTask`                                                | `dataType`               | `{ agent, task }` pair for `crew.parallel`.                             |

---

## Exported Types

### `Message`

```chuks
dataType Message {
    role: string             // "system" | "user" | "assistant" | "tool"
    content: string
    tool_calls: []ToolCall?  // set on assistant turns that invoke tools
    tool_call_id: string?    // set on role:"tool" reply turns
    name: string?            // tool name on role:"tool" reply turns
}
```

### `ToolCall`

```chuks
dataType ToolCall {
    id: string               // provider-assigned call id
    name: string             // tool name the model wants to invoke
    arguments: any           // ALREADY-PARSED JSON object (not a raw string)
}
```

### `ToolResult`

Return shape of `AgentTool.safeExecute`. When `ok = false`, `error` describes the failure.

```chuks
dataType ToolResult {
    ok: bool
    result: any              // tool output (optionally transformed)
    error: string
}
```

### `AgentStep`

One iteration of the agent loop. `tool == "none"` marks the final answer step.

```chuks
dataType AgentStep {
    stepNumber: int
    agent: string
    thought: string
    tool: string
    args: any
    result: string
    ok: bool
    durationMs: any
    tokens: int
}
```

### `AgentRunResult`

Aggregate result of one `run()` (or a crew/pipeline run). `cancelReason` is `""` on success, or one of `timeout`, `maxCost`, `llmError`, `maxSteps`, `unknownDelegate`.

```chuks
dataType AgentRunResult {
    answer: string
    structured: any          // populated when run() was given a schema; else null
    steps: []AgentStep
    totalSteps: int
    tokens: int
    inputTokens: int
    outputTokens: int
    durationMs: any
    cost: any                // USD estimate from the built-in pricing table
    model: string
    success: bool
    cancelReason: string
}
```

### `CrewTask`

One unit of work for `crew.parallel`.

```chuks
dataType CrewTask {
    agent: Agent
    task: string
}
```

### `AgentEvent`

Emitted by `Agent.stream` over a `Channel<AgentEvent>`. `type` is one of `"thought"`, `"toolCall"`, `"toolResult"`, `"answer"`. Other fields are populated based on `type`:

| `type`         | Populated fields       |
| -------------- | ---------------------- |
| `"thought"`    | `content`              |
| `"toolCall"`   | `tool`, `args`         |
| `"toolResult"` | `tool`, `result`, `ok` |
| `"answer"`     | `content`              |

```chuks
class AgentEvent {
    var type: string
    var content: string
    var tool: string
    var args: any
    var result: string
    var ok: bool
}
```

### `MemoryEntry`

```chuks
class MemoryEntry {
    var text: string
    var tags: []string
    var createdAt: any       // millis since epoch when remembered
}
```

---

## `agent.create(config)` → `Agent`

| Config                    | Type               | Default   | Description                                        |
| ------------------------- | ------------------ | --------- | -------------------------------------------------- |
| `name`                    | string             | `"agent"` | Identity for logs and multi-agent traces.          |
| `model`                   | any                | `null`    | LLM client from `chuks_ai`.                        |
| `tools`                   | `[]AgentTool`      | `[]`      | Tools the agent can use.                           |
| `delegates`               | `map[string]Agent` | `{}`      | Sub-agents — each becomes a callable tool.         |
| `systemPrompt`            | string             | `""`      | Personality and constraints.                       |
| `maxSteps`                | int                | `10`      | Max tool calls before forcing an answer.           |
| `maxTokens`               | int                | `4096`    | Max tokens per LLM response.                       |
| `temperature`             | float              | `0.2`     | LLM sampling temperature.                          |
| `timeout`                 | int                | `0`       | Cancel entire run after N ms (`0` = no limit).     |
| `toolTimeout`             | int                | `10000`   | Max time per tool call (ms).                       |
| `stepDeadlineMs`          | int                | `0`       | Max wall-clock per step (ms).                      |
| `maxCost`                 | float              | `0`       | Stop if estimated USD cost exceeds this.           |
| `retry.maxAttempts`       | int                | `3`       | LLM retry attempts on transient errors.            |
| `retry.initialMs`         | int                | `500`     | Initial backoff.                                   |
| `retry.maxMs`             | int                | `8000`    | Backoff cap.                                       |
| `retry.multiplier`        | float              | `2.0`     | Backoff multiplier.                                |
| `retry.jitterMs`          | int                | `250`     | Random jitter added per attempt.                   |
| `memory.shortTerm`        | bool               | `true`    | Keep current conversation turns.                   |
| `memory.longTerm.enabled` | bool               | `false`   | Remember across runs.                              |
| `memory.longTerm.store`   | any                | `null`    | Vector store for long-term memories.               |
| `memory.longTerm.topK`    | int                | `3`       | Relevant memories to retrieve.                     |
| `memory.maxHistory`       | int                | `20`      | Max conversation turns kept.                       |
| `allowedTools`            | `[]string`         | `[]`      | Whitelist (empty = all allowed).                   |
| `confirmBeforeCall`       | `[]string`         | `[]`      | Tool names requiring `onToolStart` approval.       |
| `onStep`                  | fn                 | `null`    | Called after every tool call with an `AgentStep`.  |
| `onThink`                 | fn                 | `null`    | Called when the LLM reasons.                       |
| `onFinish`                | fn                 | `null`    | Called with the final `AgentRunResult`.            |
| `onError`                 | fn                 | `null`    | Called on tool/agent error.                        |
| `onToolStart`             | fn                 | `null`    | Called before a tool runs; return `false` to veto. |
| `onToolEnd`               | fn                 | `null`    | Called after a tool returns.                       |

---

## Agent API

### Running

#### `run(prompt: string, contextOrOptions: any?): Task<AgentRunResult>`

The second argument may be a context object **or** an options object:

```chuks
// As context — passed to tools that read agent context.
await agent.run("Summarise the sales data", {
    "salesData": loadData(),
    "dateRange": "Q1 2026",
})

// As options — { context?, schema?, timeoutMs? }.
await agent.run("Extract fields", {
    "schema": { "name": "string", "age": "int" },
    "timeoutMs": 15000,
})
```

```chuks
var result = await agent.run("Research Chuks language", null)
println(result.answer)
println(result.totalSteps)
println(result.tokens)
println(result.durationMs)
println(result.cost)          // USD estimate
println(result.success)
println(result.cancelReason)  // "" | "timeout" | "maxSteps" | ...
println(result.steps)         // []AgentStep
```

#### `stream(prompt: string, ch: Channel<AgentEvent>, context: any?): Task<any>`

Emits live `AgentEvent`s over a typed channel.

```chuks
import { Channel } from "std/channel"
import { AgentEvent } from "pkg/@chuks/agent"

var ch: Channel<AgentEvent> = new Channel<AgentEvent>(16)
spawn researcher.stream("Research Chuks language", ch, null)

for (var evt: AgentEvent? = ch.receive(); evt != null; evt = ch.receive()) {
    if (evt.type == "thought") {
        println("· thought  : " + evt.content)
    } else if (evt.type == "toolCall") {
        println("→ call     : " + evt.tool)
    } else if (evt.type == "toolResult") {
        println("← result   : " + evt.result + " (ok=" + string(evt.ok) + ")")
    } else if (evt.type == "answer") {
        println("answer     : " + evt.content)
    }
}
```

#### `batch(tasks: []string): Task<[]AgentRunResult>`

Run the same agent on multiple prompts in parallel across CPU cores.

```chuks
var results = await researcher.batch([
    "Research Chuks language",
    "Research Go language",
    "Research Rust language",
])
```

#### `delegate(name: string, task: string): Task<AgentRunResult>`

Invoke a sub-agent registered via the `delegates` config option.

### Memory

```chuks
agent.remember("User prefers concise answers", null)
agent.remember("Building a backend in Chuks", ["context", "user"])

agent.getMemory()                  // []string
agent.getMemoryEntries()           // []MemoryEntry — text + tags + createdAt
agent.getMemoryByTag("user")       // filtered []string
agent.forget("User prefers concise answers")
agent.clearMemory()
```

### Tool management

```chuks
agent.addTool(myNewTool)
agent.removeTool("oldTool")
agent.listTools()                  // []string of registered tool names
```

### Introspection

```chuks
var last = agent.lastRun()         // AgentRunResult? — most recent run
var s    = agent.stats()
//   { totalRuns, avgSteps, avgDurationMs, totalTokens, totalCost, successRate }
println(s.totalRuns)
println(s.successRate)             // 0–100
```

---

## `tool.create(config)` → `AgentTool`

Build a tool from a function.

| Config        | Type              | Description                              |
| ------------- | ----------------- | ---------------------------------------- |
| `name`        | string            | Unique tool name.                        |
| `description` | string            | What the tool does — the LLM reads this. |
| `fn`          | fn(args)          | The handler. May be `async`.             |
| `params`      | any               | Rich param schema (see below).           |
| `validate`    | `fn(args): bool`  | Optional — pre-flight check.             |
| `transform`   | `fn(result): any` | Optional — post-process the result.      |

`params` accepts the rich shape `{ key: { type, description, required, enum, items, default } }` and is compiled to JSON Schema via `AgentTool.toSchema()`. Legacy string-valued params (`{ city: "City name" }`) are treated as `{ type: "string", required: true }`.

```chuks
var weatherTool = tool.create({
    "name":        "getWeather",
    "description": "Get current weather for a city",
    "fn":          function(args: any): any { return fetchWeatherAPI(args.city) },
    "params":      { "city": "City name, e.g. 'London' or 'Lagos'" },
    "validate":    function(args: any): bool { return args.city != null && args.city != "" },
    "transform":   function(result: any): any { return "Weather data: " + string(result) },
})
```

### `AgentTool` methods

| Method        | Signature                                  | Description                                                         |
| ------------- | ------------------------------------------ | ------------------------------------------------------------------- |
| `safeExecute` | `safeExecute(args: any): Task<ToolResult>` | Runs `validate` → `fn` → `transform` with error isolation.          |
| `execute`     | `execute(args: any): Task<any>`            | Legacy single-value form (returns `result` or error string).        |
| `toSchema`    | `toSchema(): any`                          | Convert `params` into JSON Schema (`{type, properties, required}`). |

---

## `tool.fromAgent(agent)` → `AgentTool`

Wrap an `Agent` as a tool — the foundation for the supervisor pattern.

```chuks
var researchTool = tool.fromAgent(researcher)
var analysisTool = tool.fromAgent(analyst)

var supervisor = agent.create({
    "name":         "supervisor",
    "model":        ai.anthropic("claude-sonnet-4-20250514"),
    "tools":        [researchTool, analysisTool],
    "systemPrompt": "You are a project manager. Delegate to your team.",
    "maxSteps":     20,
})

var result = await supervisor.run("Compare Chuks and Go for backends", null)
```

---

## Built-in Tools (`tools.*`)

Every factory returns an `AgentTool` you can pass straight into `agent.create({ tools: [...] })`.

| Factory                    | Signature                                | Notes                                           |
| -------------------------- | ---------------------------------------- | ----------------------------------------------- |
| `tools.webSearch(config)`  | `{ provider, apiKey, maxResults? }`      | `provider` ∈ `"brave"`, `"serper"`, `"tavily"`. |
| `tools.readURL(config)`    | `{ maxLength? }`                         | Fetch + clean HTML.                             |
| `tools.calculator()`       | `()`                                     | Safe arithmetic — no `eval`.                    |
| `tools.datetime()`         | `()`                                     | Current date/time.                              |
| `tools.httpClient(config)` | `{ allowedDomains, timeout?, methods? }` | **Domain allowlist required** for safety.       |
| `tools.fileSystem(config)` | `{ allowedPaths, allowWrite? }`          | **Path allowlist required**; sandboxed.         |
| `tools.database(config)`   | `{ connection, allowWrite?, maxRows? }`  | Read-only by default.                           |
| `tools.codeRunner(config)` | `{ language, timeout? }`                 | Execute code in a sandbox.                      |

```chuks
tools.webSearch({ "provider": "brave", "apiKey": "...", "maxResults": 5 })
tools.readURL({ "maxLength": 10000 })
tools.calculator()
tools.datetime()
tools.httpClient({ "allowedDomains": ["api.example.com"], "timeout": 5000 })
tools.fileSystem({ "allowedPaths": ["./data", "./output"], "allowWrite": true })
tools.database({ "connection": db, "allowWrite": false, "maxRows": 100 })
tools.codeRunner({ "language": "chuks", "timeout": 10000 })
```

---

## Multi-Agent Coordination

### Sequential pipelines — `crew.sequential([...])`

Agents hand off, each receiving the previous agent's `answer` as its prompt.

```chuks
import { crew } from "pkg/@chuks/agent"

var report = await crew.sequential([researcher, analyst, writer])
    .run("Write a market analysis report on Chuks vs Go")
// researcher → analyst → writer → final report
```

`crew.sequential(agents: []Agent): SequentialCrew` returns a `SequentialCrew` with one method:

```chuks
SequentialCrew.run(task: string): Task<AgentRunResult>
```

Step lists, tokens, and cost are aggregated across the whole pipeline.

### Parallel fan-out — `crew.parallel([...])`

Agents work simultaneously on separate CPU cores. Results come back in input order.

```chuks
var results = await crew.parallel([
    { "agent": researcher, "task": "Research Chuks language features" },
    { "agent": researcher, "task": "Research Go language features" },
    { "agent": analyst,    "task": "Analyse backend language trends" },
])
```

`crew.parallel(tasks: []CrewTask): Task<[]AgentRunResult>`. If one task's parent is cancelled, the rest stop too — structured concurrency.

### Supervisor pattern

Already shown — use `tool.fromAgent(...)` to expose agents as tools and let one agent delegate.

---

## Structured Concurrency — the killer feature

```chuks
async function runCampaign(topic: string): Task<string> {
    var t1 = spawn researcher.run("Research " + topic, null)
    var t2 = spawn analyst.run("Analyse "  + topic, null)
    var t3 = spawn writer.run("Draft intro for " + topic, null)

    var r1 = await t1
    var r2 = await t2
    var r3 = await t3
    return r1.answer + "\n" + r2.answer + "\n" + r3.answer
}

var task: Task<string> = spawn runCampaign("Chuks programming language")
task.timeout(30000)
var result: string = await task
```

When the timeout fires, every nested LLM call and tool execution stops — no orphaned HTTP requests, no wasted tokens.

| Framework        | When the researcher takes 35 s                                             |
| ---------------- | -------------------------------------------------------------------------- |
| Python / Node.js | Researcher keeps running, consuming credits. You pay for all tokens.       |
| **Chuks**        | All three tasks cancelled immediately. Zero wasted tokens. **Guaranteed.** |

---

## Observability

```chuks
var supportAgent = agent.create({
    "name":  "support",
    "model": ai.openai("gpt-4o"),
    "tools": [customerTool, ordersTool],
    "onStep":    function(step: any): void {
        println("[" + step.agent + "] step " + string(step.stepNumber) +
                ": "  + step.tool + " (" + string(step.durationMs) + "ms)")
    },
    "onThink":   function(thought: string): void { println("Thinking: " + thought) },
    "onFinish":  function(r: any): void {
        println("Done in " + string(r.totalSteps) + " steps, " +
                string(r.tokens) + " tokens, $" + string(r.cost))
    },
    "onError":   function(err: any): void { println("Error: " + err.message) },
})
```

---

## Full Example — Support Agent Server

```chuks
import { agent, tool, tools } from "pkg/@chuks/agent"
import { ai } from "pkg/@chuks/ai"
import { http } from "std/http"
import { json } from "std/json"
import { dotenv } from "std/dotenv"

dotenv.load()

var customerTool = tool.create({
    "name":        "getCustomer",
    "description": "Look up a customer by ID or email",
    "fn": async function(args: any): Task<any> {
        var rows = await db.query(
            "SELECT * FROM customers WHERE id=? OR email=?",
            args.identifier, args.identifier
        )
        return json.stringify(rows)
    },
    "params": { "identifier": "Customer ID or email address" },
})

var supportAgent = agent.create({
    "name":         "support",
    "model":        ai.anthropic("claude-sonnet-4-20250514"),
    "tools":        [customerTool, tools.webSearch({ "provider": "brave" }), tools.calculator()],
    "systemPrompt": "You are a customer support agent. Always look up the customer before answering. Be empathetic and professional.",
    "maxSteps":     8,
    "timeout":      20000,
    "maxCost":      0.05,
    "temperature":  0.3,
    "onStep": function(step: any): void {
        println("[support] " + step.tool + " → " + string(step.durationMs) + "ms")
    },
})

http.post("/support", async function(req: any, res: any): Task<void> {
    var body = json.parse(req.body)
    var r    = await supportAgent.run(body.question, null)
    res.json({
        "answer": r.answer,
        "steps":  r.totalSteps,
        "tokens": r.tokens,
        "cost":   r.cost,
    })
})

http.listen(8080)
println("Support agent on :8080")
```

---

## Why chuks_agent

| Feature                 | LangChain | AutoGen | CrewAI | **chuks_agent**            |
| ----------------------- | --------- | ------- | ------ | -------------------------- |
| Type safety             | No        | No      | No     | **Compile-time**           |
| Guaranteed cancellation | No        | No      | No     | **Structured concurrency** |
| Real parallel agents    | No        | No      | No     | **`spawn` (CPU cores)**    |
| Single binary deploy    | No        | No      | No     | **Yes**                    |
| Built-in cost limits    | No        | Partial | No     | **`maxCost`**              |
| Streaming via channels  | No        | No      | No     | **`Channel<AgentEvent>`**  |
| Tagged memory built-in  | Add-on    | Add-on  | Add-on | **Config option**          |

---

## License

MIT
