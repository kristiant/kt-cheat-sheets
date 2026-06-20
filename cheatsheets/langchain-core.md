# LangChain Core — Internals

**What it is:** The internals of `@langchain/core` — the foundational package (messages, runnables, prompts, tools, output parsers) every LangChain package builds on.

**Why people use it:** Understanding the core types makes custom chains, streaming, and tool loops debuggable — under every abstraction you're really manipulating `BaseMessage[]` flowing through `Runnable`s.

**Typically used for:** Custom message handling, streaming accumulation, tool-call loops, and provider integrations.

> Usage-level LangChain is in [langchain.md](langchain.md); this sheet is the internals. Source: `@langchain/core` (`langchain-core/src`).

---

## Messages

A conversation is a `BaseMessage[]`. Every concrete message type extends `BaseMessage` and maps to an LLM role.

### Message types (`messages/`)

| Class | Role | Key extra fields |
|---|---|---|
| `HumanMessage` | user / input | — |
| `AIMessage` | model / output | `tool_calls`, `invalid_tool_calls`, `usage_metadata` |
| `SystemMessage` | system / prompt | — |
| `ToolMessage` | tool / result | `tool_call_id`, `status`, `artifact` |
| `FunctionMessage` | legacy OpenAI function-calling | **deprecated — avoid** |
| `ChatMessage` | arbitrary role | `role: string` |

### MessageContent (`base.ts`)

```ts
type MessageContent = string | ContentBlock[];
```

Either a plain string, or a typed block array for multimodal/structured content.

### ContentBlock (`content/`)

Typed blocks a `MessageContent` array can hold:

```ts
{ type: "text", text: string }
{ type: "reasoning", ... }          // model chain-of-thought
{ type: "image" | "audio" | ... }   // multimodal
{ type: "tool_use" | "tool_result", ... }
```

### `tool_calls` on AIMessage — the tool round-trip

When the model decides to call a tool, it returns an `AIMessage` carrying structured `tool_calls` (and `invalid_tool_calls` for parse failures). You run each tool and append a `ToolMessage` with a matching `tool_call_id`.

```ts
const ai = await model.invoke(messages);     // AIMessage
for (const call of ai.tool_calls ?? []) {
  const result = await tools[call.name].invoke(call.args);
  messages.push(new ToolMessage({ content: result, tool_call_id: call.id }));
}
// loop back to the model with the appended ToolMessages
```

The `tool_call_id` is the join key — it's how the model pairs each result with the call it made.

### Streaming — `*Chunk` classes

Every message type has a `*Chunk` counterpart (`AIMessageChunk`, `HumanMessageChunk`, …) produced by `.stream()`. Chunks are **additive** — concatenating them reconstructs the full message:

```ts
let acc: AIMessageChunk | undefined;
for await (const chunk of await model.stream(input)) {
  acc = acc ? acc.concat(chunk) : chunk;     // chunks merge via mergeContent
}
// acc is now the complete AIMessage(Chunk)
```

### `mergeContent` / `_mergeDicts` (`base.ts`)

The merge utilities that power chunk accumulation — `mergeContent` concatenates content (string or block arrays), `_mergeDicts` deep-merges `additional_kwargs`/metadata. Understand these when building custom streaming chains, since `concat` on chunks delegates to them.

### RemoveMessage (`modifier.ts`)

A marker — not a real conversational message — used in LangGraph state to **delete a message from history by id**.

```ts
import { RemoveMessage } from "@langchain/core/messages";
return { messages: [new RemoveMessage({ id: staleMessageId })] };   // graph reducer drops it
```

### Mental model

```
human sends      → HumanMessage
model responds   → AIMessage
  └─ if tool_calls present:
       run each tool → append ToolMessage[] (matched by tool_call_id)
       → loop back to the model
```

A conversation is just `BaseMessage[]` growing over time; streaming yields `*Chunk`s that merge back into those messages.

---

## Runnables

`Runnable<RunInput, RunOutput>` (`base.ts`) is the base abstraction **every** LangChain component implements — models, prompts, parsers, retrievers, chains. Three execution methods:

| Method | Description |
|---|---|
| `.invoke(input)` | single call → single output |
| `.batch(inputs[])` | parallel calls → array of outputs |
| `.stream(input)` | async generator of chunks |

### `.pipe()` — LCEL composition

Chains runnables sequentially — the output of one becomes the input of the next. Returns a `RunnableSequence`, itself a Runnable, so chains nest freely.

```ts
const chain = prompt.pipe(model).pipe(outputParser);
// RunnableSequence<PromptInput, ParsedOutput>
```

### RunnableAssign — augment a dict (vs `pipe` replacing it)

`.pipe()` *replaces* the value at each step; `RunnableAssign` *merges* new keys into the existing dict. The idiomatic entry point is the static `RunnablePassthrough.assign(...)` (sugar for `RunnablePassthrough` + `RunnableMap`):

```ts
const chain = RunnablePassthrough.assign({
  summary: summaryChain,      // both run on the ORIGINAL input, in parallel
  sentiment: sentimentChain,  // results merged back under these keys
});
// → { ...input, summary, sentiment }
```

Use it to *enrich* a dict as it flows — retrieved context, computed fields, intermediate results later steps need — without losing earlier keys. (See `pipe` vs `assign` in [langchain.md](langchain.md).)

### RunnableEach — map over a list

`.map()` returns a `RunnableEach` that applies a runnable to each element of an array input.

```ts
const chain = someRunnable.map();   // T[] → U[]
```

### RunnablePick — pluck keys (inverse of Assign)

```ts
const pick = new RunnablePick(["name", "city"]);   // or someRunnable.pick(["name", "city"])
// { name: "Alice", city: "NY", age: 30 } → { name: "Alice", city: "NY" }
```

### RouterRunnable — key-based dispatch

```ts
const router = new RouterRunnable({ runnables: { upper, reverse } });
router.invoke({ key: "upper", input: "hello" });   // → "HELLO"
```

Less common than `RunnableBranch` (condition-based routing); use for explicit key dispatch.

### RunnableWithMessageHistory — deprecated

Wrapped a chain to auto-load/save chat history per session via a `getMessageHistory` callback keyed by session id. **Deprecated** — LangGraph's checkpointer persistence replaces it. Recognise it in older code; don't reach for it in new.

---

## Prompts

`BasePromptTemplate` (`base.ts`) extends `Runnable` — all prompts are `Runnable<InputValues, PromptValue>`. Invoking a prompt yields a **`PromptValue`** (not a string or messages directly), convertible via `.toString()` / `.toChatMessages()`. Shared fields: `inputVariables`, `partialVariables`, `outputParser`.

### `PromptTemplate` — string prompts (LLMs)

```ts
const p = PromptTemplate.fromTemplate("Tell me about {topic}");
// inputVariables inferred as ["topic"] from the template
```

Supports `f-string` (default, `{var}`) and `mustache` (`{{var}}`) formats.

### `ChatPromptTemplate` — chat models (the common one)

```ts
ChatPromptTemplate.fromMessages([
  ["system", "You are {role}"],
  ["human", "{input}"],
]);
```

Each tuple is coerced into a `*MessagePromptTemplate`. Use the static `fromMessages` — not the constructor.

### Per-role message templates

`HumanMessagePromptTemplate`, `AIMessagePromptTemplate`, `SystemMessagePromptTemplate`, `ChatMessagePromptTemplate` (arbitrary role) — used inside `fromMessages` or standalone.

### `MessagesPlaceholder` — inject a message array

A named slot for a `BaseMessage[]` — essential for chat history:

```ts
ChatPromptTemplate.fromMessages([
  ["system", "You are helpful"],
  new MessagesPlaceholder("history"),     // optional: true → inject nothing if missing
  ["human", "{input}"],
]);
// invoke with { history: BaseMessage[], input: "..." }
```

### `partial()` — pre-fill variables

```ts
const partial = await prompt.partial({ role: "an expert chef" });
// now only needs { input } at invoke time
```

`partialVariables` can be **async functions**, evaluated at invoke time — handy for dynamic values (current date, feature flags).

### Few-shot — `FewShotPromptTemplate` / `FewShotChatMessagePromptTemplate`

Injects examples, either a static `examples` list or a dynamic `exampleSelector` (e.g. semantic-similarity selection):

```ts
new FewShotPromptTemplate({
  examples,
  examplePrompt,                  // PromptTemplate formatting each example
  suffix: "Q: {input}\nA:",
  inputVariables: ["input"],
});
```

### Other

- **`PipelinePromptTemplate`** — compose prompts; each sub-prompt's output feeds a variable into the final prompt. Niche, for reusable prompt parts.
- **`StructuredPrompt`** — a `ChatPromptTemplate` carrying a JSON schema; piped to a model it auto-wires `.withStructuredOutput()`, so the schema travels with the prompt.
- **Template formats** — `f-string` (default, `{var}`) and `mustache` (`{{var}}`, supports loops/conditionals).

### Mental model

```ts
ChatPromptTemplate.fromMessages([...])
  .pipe(chatModel)        // PromptValue → AIMessage
  .pipe(outputParser);    // AIMessage → structured output
```

Prompts are the entry point of every LCEL chain — they validate variables at construction and produce typed `PromptValue`s that models consume.

---

## Language models

### Class hierarchy

```
Runnable
 └─ BaseLangChain
     └─ BaseLanguageModel
         ├─ BaseLLM         text in → text out (legacy)
         └─ BaseChatModel   messages in → message out
             └─ SimpleChatModel   convenience subclass
```

`BaseLLM` is legacy; everything modern uses `BaseChatModel`.

### Input — `BaseLanguageModelInput`

```ts
type BaseLanguageModelInput =
  | BasePromptValueInterface   // from a prompt template
  | string                     // convenience
  | BaseMessageLike[];         // array of messages
```

All three are normalised to `BaseMessage[]` internally before `_generate`.

### `BaseChatModel` — what a custom provider implements

Two abstract methods; the base class layers `invoke`/`stream`/`batch`, caching, callbacks, and retry on top:

- `_generate(messages, options, runManager)` — single call
- `_streamResponseChunks(...)` — streaming; yields `AsyncGenerator<ChatGenerationChunk>`

`SimpleChatModel` simplifies further — implement just `_call(messages) → string`.

### `bindTools()`

Binds tool definitions to the model, returning a new runnable. Each provider implements it differently; output is still an `AIMessage`, now with `tool_calls` populated.

```ts
const modelWithTools = model.bindTools([tool1, tool2]);
```

### `withStructuredOutput()`

High-level structured output — give it a Zod/JSON schema, get a runnable returning a typed object. Internally wires `bindTools` + a parsing runnable.

```ts
const structured = model.withStructuredOutput(z.object({ name: z.string() }));
await structured.invoke("What is your name?");   // → { name: "..." }
```

- `method`: `"functionCalling"` (default) · `"jsonMode"` (model returns JSON, parsed) · `"jsonSchema"` (provider-native schema mode, OpenAI/Gemini).
- `includeRaw: true` → `{ raw: AIMessage, parsed: T }` when you need both.

### Other model internals

- **Caching** — `new ChatOpenAI({ cache: new InMemoryCache() })` (or `cache: true` for the global singleton). Key = prompt content + model params; hits skip the API call.
- **`outputVersion`** — `AIMessage.content` format: `"v0"` (default, provider-raw) vs `"v1"` (normalised `ContentBlock[]`). Set via constructor or `LC_OUTPUT_VERSION`. Matters for provider-agnostic code that inspects content.
- **Call options** (`BaseChatModelCallOptions`) — per-call: `tool_choice` (`"auto"`/`"any"`/`"none"`/name), `stop: string[]`, `signal: AbortSignal`, `timeout`.
- **Streaming internals** — `_streamResponseChunks` is the legacy bridge; the newer protocol emits typed `ChatModelStreamEvent`s (`content_block_start`/`_delta`/`_stop`, matching Anthropic's model). A `ReplayBuffer` lets multiple consumers iterate the same stream.

### Mental model

```ts
prompt.pipe(model).pipe(outputParser);
// model.invoke():
//   _generateCached()   → cache hit? return early
//   _generateUncached() → _generate() (or bridges _streamResponseChunks)
//   → wraps result in AIMessage, fires callbacks
```

Building a custom provider = implement `_generate` + `_streamResponseChunks` + `bindTools`; everything else is inherited.
