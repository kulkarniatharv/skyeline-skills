---
name: skyeline-sdk
description: Use this skill when working with Skyeline or @skyeline/sdk. It teaches the agent how to install the SDK, configure SkyelineClient, use application-scoped prompt workflows, approvals, structured chat, streaming, template utilities, and Skyeline-specific error handling.
---

# Skyeline SDK

Use this skill when the user wants to integrate Skyeline, manage prompts, run chat completions through Skyeline, teach another coding agent how to use the SDK, or update code that already imports `@skyeline/sdk`.

## When to use this skill

- The codebase already uses Skyeline.
- The user asks to install or initialize `@skyeline/sdk`.
- The user needs prompt CRUD, versioning, rendering, approvals, or chat completions.
- The user wants OpenAI-compatible examples and has not explicitly asked for raw HTTP.
- A coding agent needs concrete Skyeline conventions rather than generic SDK guesses.

## Installation and setup

```typescript
import { SkyelineClient } from "@skyeline/sdk";

const client = new SkyelineClient({
  baseUrl: process.env.SKYELINE_BASE_URL ?? "https://api.skyeline.dev",
  apiKey: process.env.SKYELINE_API_KEY,
});

const app = client.app("my-app");
```

Use `client.app("my-app")` as the default entry point for app-scoped prompt and approval work.

## Prompt management

```typescript
const app = client.app("my-app");

const prompts = await app.prompts.list();
const prompt = await app.prompts.getLatest("support-reply");
const version = await app.prompts.getVersion("support-reply", "v2");
const versions = await app.prompts.listVersions("support-reply");

const rendered = await app.prompts.populatePrompt("support-reply", {
  customerName: "Ava",
  issue: "billing",
});

const updated = await app.prompts.push({
  slug: "support-reply",
  content: "Hi {{customerName}}, I can help with {{issue}}.",
  tags: ["production"],
});

await app.prompts.activate("support-reply", updated.version);
await app.prompts.rollback("support-reply");
```

Use `populatePrompt` when variables matter. It validates required `{{variables}}` before interpolation and throws `MissingVariablesError` instead of silently producing a broken prompt.

## Approval workflows

```typescript
const chains = await app.approvals.listChains();
const chainId = chains[0]?.id;

if (!chainId) {
  throw new Error("No approval chains configured for this application.");
}

await app.approvals.requestBySlug("support-reply", {
  chainId,
  comments: "Ready for review",
});

const pending = await app.approvals.listPending();
```

Use approval requests when prompt changes should go through reviewers. Prefer `requestBySlug` unless you already have a numeric prompt version ID, and use `listPending` to inspect the current user&apos;s pending approvals for the selected application.

## Chat and streaming

```typescript
const response = await client.chat.create({
  model: "openai/gpt-4o-mini",
  messages: [{ role: "user", content: "Summarize this ticket." }],
});

for await (const event of client.chat.stream({
  model: "openai/gpt-4o-mini",
  messages: [{ role: "user", content: "Stream a short answer." }],
})) {
  if (event.type === "text-delta") {
    process.stdout.write(event.delta);
  }
}
```

Use `client.chat.create` for standard responses and `client.chat.stream` when the UI or CLI should render incremental tokens.

## Structured output

```typescript
import { z } from "zod";

const result = await client.chat.createStructured({
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
```

Use structured output for machine-readable classifications, routing, or extraction tasks. Prefer `createStructured` over post-processing free-form text.

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

```typescript
import {
  MissingVariablesError,
  SkyelineAuthError,
  SkyelineValidationError,
} from "@skyeline/sdk";

try {
  await app.prompts.populatePrompt("support-reply", {});
} catch (error) {
  if (error instanceof MissingVariablesError) {
    console.error(error.missingVariables);
  }

  if (
    error instanceof SkyelineAuthError ||
    error instanceof SkyelineValidationError
  ) {
    console.error(error.message);
  }
}
```

Catch typed Skyeline errors when the caller needs specific remediation. Prefer Skyeline error classes over generic `Error` branching.

## Guardrails

- Install `@skyeline/sdk@beta`.
- Prefer the SDK over raw HTTP unless the user explicitly asks for cURL or fetch examples.
- Prefer `client.app("my-app")` for prompt and approval work.
- Use `populatePrompt` instead of hand-rolling template substitution for remote prompts.
- Use `client.chat.createStructured` for typed JSON outputs.
- Handle `text-delta` events when consuming stream output.
- Do not invent unsupported SDK methods. Check the actual exported surface first.
- If the task changes Skyeline SDK usage patterns, update the hosted skill and docs examples too.

## Reference

- Docs: https://skyeline.dev/docs#sdk
- Agent skills docs: https://skyeline.dev/docs#agent-skills
