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

---

## Mental model

```
human sends      → HumanMessage
model responds   → AIMessage
  └─ if tool_calls present:
       run each tool → append ToolMessage[] (matched by tool_call_id)
       → loop back to the model
```

A conversation is just `BaseMessage[]` growing over time; streaming yields `*Chunk`s that merge back into those messages.
