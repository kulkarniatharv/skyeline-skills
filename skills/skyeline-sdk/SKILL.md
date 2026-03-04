---
name: skyeline-sdk
description: Use this skill when working with Skyeline or @skyeline/sdk. It teaches the agent how to install the SDK, configure SkyelineClient, use application-scoped prompt workflows, approvals, chat completions, structured output, webhook verification, template utilities, and Skyeline-specific error handling.
---

# Skyeline SDK

Use this skill when the user wants to integrate Skyeline, manage prompts, run chat completions through Skyeline, verify webhook events, teach another coding agent how to use the SDK, or update code that already imports `@skyeline/sdk`.

## When to use this skill

- The codebase already uses Skyeline.
- The user asks to install or initialize `@skyeline/sdk`.
- The user needs prompt CRUD, versioning, rendering, approvals, or chat completions.
- The user wants to verify Skyeline webhook events.
- The user wants OpenAI-compatible examples and has not explicitly asked for raw HTTP.
- A coding agent needs concrete Skyeline conventions rather than generic SDK guesses.

## Installation and setup

```typescript
import { SkyelineClient } from "@skyeline/sdk";

const client = new SkyelineClient({
  baseUrl: process.env.SKYELINE_BASE_URL ?? "https://api.skyeline.dev",
  apiKey: process.env.SKYELINE_API_KEY,
  // Optional:
  headers: { "X-Custom": "value" },   // additional headers for all requests
  timeoutMs: 600_000,                  // default request timeout (10 min)
  maxRetries: 2,                       // default retry count (3 total attempts)
  fetchApi: customFetch,               // custom fetch implementation
});

const app = client.app("my-app");
```

`baseUrl` and `apiKey` accept `string | undefined` — pass env vars directly; the constructor throws a descriptive error if either is missing. Use `client.app("my-app")` as the default entry point for app-scoped prompt and approval work.

## Prompt management

```typescript
const app = client.app("my-app");

// List and fetch
const prompts = await app.prompts.list();
const filtered = await app.prompts.list({ status: "active" });
const prompt = await app.prompts.getLatest("support-reply");
const version = await app.prompts.getVersion("support-reply", "v2");
const versions = await app.prompts.listVersions("support-reply");

// Render with variable validation
const rendered = await app.prompts.populatePrompt("support-reply", {
  customerName: "Ava",
  issue: "billing",
});
// rendered.content = interpolated string
// rendered.prompt  = original PromptVersion

// Render a specific version
const v2Rendered = await app.prompts.populatePrompt(
  "support-reply",
  { customerName: "Ava", issue: "billing" },
  { version: "v2" },
);

// Push a new version
const updated = await app.prompts.push({
  slug: "support-reply",
  content: "Hi {{customerName}}, I can help with {{issue}}.",
  tags: ["production"],
});

// Lifecycle
await app.prompts.activate("support-reply", updated.version);
await app.prompts.promote("support-reply", "v3"); // alias for activate
await app.prompts.archive("support-reply", "v1");
await app.prompts.rollback("support-reply");
await app.prompts.delete("support-reply");

// Tags
const tags = await app.prompts.listTags();
await app.prompts.updateTags("support-reply", {
  tags: ["production", "v2"],
});
```

Use `populatePrompt` when variables matter. It validates required `{{variables}}` before interpolation and throws `MissingVariablesError` instead of silently producing a broken prompt.

## Approval workflows

```typescript
// Discover chains
const chains = await app.approvals.listChains();
const chainId = chains[0]?.id;

if (!chainId) {
  throw new Error("No approval chains configured for this application.");
}

// Request approval (auto-resolves promptId from latest version)
await app.approvals.requestBySlug("support-reply", {
  chainId,
  comments: "Ready for review",
});

// Request approval for a specific version
await app.approvals.requestBySlug(
  "support-reply",
  { chainId },
  { version: "v2" },
);

// List pending approvals for this application
const pending = await app.approvals.listPending();

// View approval history
const history = await app.approvals.historyBySlug("support-reply");

// Approve or reject
await app.approvals.respond(requestId, {
  action: "approved",
  comments: "Looks good!",
});

// Cancel a pending request
await app.approvals.cancel(requestId);

// Fetch a specific request by ID
const request = await app.approvals.get(requestId);
```

Use `requestBySlug` unless you already have a numeric prompt version ID. Use `listPending` to inspect the current application&apos;s pending approvals.

## Chat and streaming

```typescript
// Non-streaming
const response = await client.chat.create({
  model: "openai/gpt-4o-mini",
  messages: [{ role: "user", content: "Summarize this ticket." }],
});
console.log(response.content);      // text response
console.log(response.finishReason); // "stop" | "length" | "tool_calls" | ...
console.log(response.usage);        // { inputTokens, outputTokens, totalTokens }

// Streaming
for await (const event of client.chat.stream({
  model: "openai/gpt-4o-mini",
  messages: [{ role: "user", content: "Stream a short answer." }],
})) {
  if (event.type === "text-delta") {
    process.stdout.write(event.delta);
  }
}

// Tool calls
const toolResponse = await client.chat.create({
  model: "openai/gpt-4o",
  messages: [{ role: "user", content: "What is the weather in SF?" }],
  tools: [
    {
      name: "get_weather",
      description: "Get weather for a city",
      parameters: {
        type: "object",
        properties: { city: { type: "string" } },
        required: ["city"],
      },
    },
  ],
});

for (const call of toolResponse.toolCalls) {
  console.log(call.name, call.arguments);
}

// Per-request overrides (timeout, retries, headers)
await client.chat.create(
  { model: "openai/gpt-4o-mini", messages: [{ role: "user", content: "Hi" }] },
  { timeoutMs: 5000, maxRetries: 0, headers: { "X-Trace-Id": "trace-123" } },
);
```

Use `client.chat.create` for standard responses and `client.chat.stream` when the UI or CLI should render incremental tokens. Stream events are typed: `text-delta`, `tool-call`, `tool-result`, `finish`, `error`.

## Structured output

```typescript
import { z } from "zod";

// object — returns inferred Zod type
const classification = await client.chat.createStructured({
  model: "openai/gpt-4o-mini",
  messages: [{ role: "user", content: "Classify this support issue." }],
  output: {
    type: "object",
    name: "support_classification",
    schema: z.object({
      priority: z.enum(["low", "medium", "high"]),
      team: z.enum(["billing", "support", "sales"]),
    }),
  },
});

// array — returns inferred Zod type[]
const items = await client.chat.createStructured({
  model: "openai/gpt-4o-mini",
  messages: [{ role: "user", content: "Return 3 tags for: distributed tracing" }],
  output: {
    type: "array",
    elementSchema: z.object({ tag: z.string() }),
  },
});

// choice — returns one of the string options
const route = await client.chat.createStructured({
  model: "openai/gpt-4o-mini",
  messages: [{ role: "user", content: "Route to: sales, support, or engineering?" }],
  output: {
    type: "choice",
    options: ["sales", "support", "engineering"],
  },
});

// json — returns unknown (untyped JSON)
const freeform = await client.chat.createStructured({
  model: "openai/gpt-4o-mini",
  messages: [{ role: "user", content: "Return a JSON summary." }],
  output: { type: "json" },
});
```

`createStructured` is non-streaming only. Use `object` with Zod for typed results, `array` for lists, `choice` for enum routing, and `json` for untyped JSON.

## Webhook verification

```typescript
import {
  SkyelineClient,
  SkyelineWebhookVerificationError,
} from "@skyeline/sdk";

export async function handleWebhook(request: Request) {
  const rawBody = await request.text();

  try {
    const event = client.webhooks.constructEvent(
      rawBody,
      request.headers,
      process.env.SKYELINE_WEBHOOK_SECRET!,
    );

    switch (event.type) {
      case "approval.requested":
        console.log(event.data.promptSlug, event.data.chainName);
        break;
      case "approval.updated":
        console.log(event.data.action, event.data.actorId);
        break;
      case "approval.completed":
        console.log(event.data.status, event.data.promptVersion);
        break;
    }
  } catch (error) {
    if (error instanceof SkyelineWebhookVerificationError) {
      return new Response(`Invalid webhook: ${error.code}`, { status: 400 });
    }
    return new Response("Unexpected error", { status: 500 });
  }

  return new Response("ok", { status: 200 });
}
```

`constructEvent` verifies `X-Skyeline-Signature` / `X-Skyeline-Timestamp` headers and parses a typed event. Use `verifySignature` if you only need signature verification without payload parsing. Webhook event types: `approval.requested`, `approval.updated`, `approval.completed`.

## Offline template utilities

```typescript
import { extractVariables, renderTemplate } from "@skyeline/sdk";

const variables = extractVariables("Hi {{customerName}}, issue: {{issue}}");
const content = renderTemplate("Hi {{customerName}}", {
  customerName: "Ava",
});
```

Use these utilities when you need local validation or rendering without an API call.

## Error handling

All API errors extend `SkyelineApiError`. Catch specific subclasses for targeted remediation.

```typescript
import {
  SkyelineApiError,
  SkyelineAuthError,
  SkyelineNotFoundError,
  SkyelineRateLimitError,
  SkyelineValidationError,
  SkyelineWebhookVerificationError,
  MissingVariablesError,
} from "@skyeline/sdk";

try {
  await app.prompts.getLatest("nonexistent");
} catch (error) {
  if (error instanceof SkyelineNotFoundError) {
    // 404 — resource does not exist
    console.error(error.message, error.status, error.code);
  }

  if (error instanceof SkyelineAuthError) {
    // 401 or 403 — invalid/expired API key or insufficient permissions
  }

  if (error instanceof SkyelineRateLimitError) {
    // 429 — too many requests; use retryAfterMs for backoff
    console.error(error.retryAfterMs);
  }

  if (error instanceof SkyelineValidationError) {
    // 400 or 422 — invalid request parameters
  }

  if (error instanceof SkyelineApiError) {
    // Catch-all for any API error (all above extend this)
    console.error(error.message, error.status, error.code, error.type);
  }
}

// Template-specific error
try {
  await app.prompts.populatePrompt("support-reply", {});
} catch (error) {
  if (error instanceof MissingVariablesError) {
    console.error(error.missingVariables); // ["customerName", "issue"]
  }
}
```

| Error Class | HTTP Status | When |
|---|---|---|
| `SkyelineAuthError` | 401, 403 | Invalid API key or insufficient permissions |
| `SkyelineNotFoundError` | 404 | Resource does not exist |
| `SkyelineRateLimitError` | 429 | Too many requests (`retryAfterMs` for backoff) |
| `SkyelineValidationError` | 400, 422 | Invalid request parameters |
| `SkyelineApiError` | Any other | Server errors, unexpected failures (catch-all base) |
| `SkyelineWebhookVerificationError` | — | Invalid webhook signature, timestamp, or payload |
| `MissingVariablesError` | — | `populatePrompt` called without required variables |

## OpenAI compatibility

Skyeline exposes OpenAI-compatible endpoints. Use the OpenAI SDK directly when you already have OpenAI client code:

```typescript
import OpenAI from "openai";

const openai = new OpenAI({
  apiKey: process.env.SKYELINE_API_KEY,
  baseURL: process.env.SKYELINE_BASE_URL,
});

const completion = await openai.chat.completions.create({
  model: "gpt-4o-mini",
  messages: [{ role: "user", content: "Hello!" }],
});
```

## Guardrails

- Install `@skyeline/sdk@beta`.
- Prefer the SDK over raw HTTP unless the user explicitly asks for cURL or fetch examples.
- Prefer `client.app("my-app")` for prompt and approval work.
- Use `populatePrompt` instead of hand-rolling template substitution for remote prompts.
- Use `client.chat.createStructured` for typed JSON outputs — supports `object`, `array`, `choice`, and `json` types.
- Handle `text-delta` events when consuming stream output.
- Catch `SkyelineRateLimitError` and use `retryAfterMs` for backoff when building resilient integrations.
- Use `client.webhooks.constructEvent` for webhook verification — never manually parse or verify signatures.
- Do not invent unsupported SDK methods. The complete method surface is documented in the sections above.
- If the task changes Skyeline SDK usage patterns, update the hosted skill and docs examples too.

## Reference

- Docs: https://skyeline.dev/docs/sdk
- Agent skills docs: https://skyeline.dev/docs/agent-skills
