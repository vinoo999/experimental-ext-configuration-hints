# A2A Protocol Extension: Configuration Hints (v1 Draft)

- **URI:** `https://a2a-protocol.org/extensions/configuration-hints/v1`
- **Type:** Profile Extension
- **Version:** 1.0.0 (draft)
- **Authors:** Vinay Ramesh (Bloomberg)

## 1. Abstract

Model APIs let the caller choose the model, the reasoning effort, and the output length for each
call. An agent that delegates work to another agent over A2A has no standard way to express the
same choices, even when it knows whether a subtask is a quick lookup or a hard, open-ended
problem.

This extension lets an agent declare, on its Agent Card, a JSON Schema of the harness settings a
client may hint. A few property names have shared semantics. A client sends hint values in
`SendMessageRequest.metadata`, keyed by the extension URI. Hints are not commands: when the agent
cannot honor a hint, it proceeds with its defaults instead of failing. A client can opt in to
strict handling for specific hints.

### 1.1 Definitions

- **Harness:** The parts of an agent that decide how a request is processed: models, reasoning
  settings, iteration limits, and the tools and instructions available to the model.
- **Configuration schema:** The JSON Schema in the Agent Card that describes the hints the agent
  accepts.
- **Hint request:** The object a client sends under the extension URI in
  `SendMessageRequest.metadata`.

### 1.2 Non-Goals

Reporting the applied configuration or usage data back to the client is out of scope for this
extension.

## 2. Extension URI

- **URI:** `https://a2a-protocol.org/extensions/configuration-hints/v1`
- **Constraint:** This is the only URI accepted for this version of the extension.

The URI is an identifier. HTTP access to it is not expected.

## 3. Extension Capability Declaration

An agent that accepts hints MUST declare the extension in `AgentCard.capabilities.extensions`
with a `configurationSchema` in `params`.

```json
{
  "uri": "https://a2a-protocol.org/extensions/configuration-hints/v1",
  "description": "Accepts hints for reasoning effort, model, and output length.",
  "required": false,
  "params": {
    "configurationSchema": {
      "$schema": "https://json-schema.org/draft/2020-12/schema",
      "type": "object",
      "properties": {
        "reasoningEffort": { "type": "string", "enum": ["low", "medium", "high"], "default": "medium" },
        "model": { "type": "string", "enum": ["research-fast", "research-deep"], "default": "research-fast" },
        "maxOutputTokens": { "type": "integer", "minimum": 256, "maximum": 32000 },
        "citationStyle": { "type": "string", "enum": ["apa", "ieee"], "description": "Citation format." }
      },
      "additionalProperties": false
    }
  }
}
```

| Field | Type | Required | Description |
| :---- | :--- | :------- | :---------- |
| `configurationSchema` | object | Yes | JSON Schema 2020-12 for the `hints` object (Section 4.2). MUST have `"type": "object"`. Any `$ref` MUST resolve within the schema. |

Agents SHOULD set `"additionalProperties": false`, SHOULD give each property a `default` where
one exists, and SHOULD give agent-specific properties a `description`. The schema MAY use the
top-level `required` keyword to say which hints the agent expects. A missing expected hint does
not fail the request; the agent uses its default.

### 3.1 Well-Known Properties

An agent need not accept any of these. If its schema contains one of these names, the property
MUST have the semantics below.

| Property | Type | Semantics |
| :------- | :--- | :-------- |
| `reasoningEffort` | string | Reasoning work for the request. The subschema MUST use `enum`, ordered from least to most effort. Agents SHOULD pick labels from `none`, `minimal`, `low`, `medium`, `high`, `max`. Effort applies to the whole harness; how a level maps to model settings, passes, or subagents is not exposed. |
| `model` | string | The agent's primary model for the request. The subschema MUST list the allowed values with `enum` or `oneOf` of `const`. Values are opaque identifiers chosen by the agent. |
| `maxOutputTokens` | integer | Upper bound on tokens in the content returned for the request. |
| `maxTotalTokens` | integer | Upper bound on all model tokens the agent consumes for the request. |
| `maxDurationSeconds` | number | Target upper bound on wall-clock processing time. |
| `tools` | array | Client-supplied tool references (Section 6). |
| `agentSkills` | array | Client-supplied Agent Skills references (Section 6). |

Token counts use the agent's own tokenizer and are approximate. When a limit is reached, the
agent SHOULD return the best result it has.

Other properties (for example `citationStyle`, or `temperature`) are allowed. Their meaning comes
from the schema's `description`.

### 3.2 Extended Agent Card

An agent MAY advertise a reduced schema, or omit the extension, on its public Agent Card and a
fuller schema on its extended Agent Card. Options that reveal vendor relationships or widen the
attack surface (`model`, `tools`, `agentSkills`) SHOULD appear only on the extended Agent Card.
Advertising a value is not authorization. The agent MUST check at request time that the
authenticated client may use each hinted value.

## 4. Client Extension Request

### 4.1 Extension Activation

Agents SHOULD declare the extension with `"required": false`. A client that sends hints MUST
include the extension URI in the `A2A-Extensions` service parameter. If a request carries a hint
request but the client did not activate the extension, the agent SHOULD ignore it and MUST NOT
reject the request because of it.

### 4.2 Wire Format

The hint request is stored at `SendMessageRequest.metadata["<extension URI>"]`.

| Field | Type | Required | Description |
| :---- | :--- | :------- | :---------- |
| `hints` | object | Yes | Hint values, keyed by property names from the configuration schema. |
| `required` | array of strings | No | Names of properties in `hints` the client needs honored exactly (Section 5.2). Default: empty. |

`SendMessageRequest.metadata` is used instead of `Message.metadata` because hints apply to one
request, like the fields of `SendMessageConfiguration`. `Message.metadata` is stored in
`Task.history` and would suggest the hints still apply on later turns. The companion Output Schema
extension uses the same location.

### 4.3 Scope Across Turns

- A hint request applies to the `SendMessageRequest` it is sent with.
- An agent MAY keep applying it on later turns of the same Task, for example after
  `TASK_STATE_INPUT_REQUIRED`. A hint request on a later turn replaces any earlier one; the two
  are not merged.
- A client that needs the same configuration on a later turn SHOULD send it again.
- A new Task does not inherit hints from another Task, even in the same `contextId`.
- An agent that keeps hints SHOULD discard them when the Task reaches a terminal state.

## 5. Server Behavior

### 5.1 Hints, Not Commands

The agent owns its harness. It validates each hint against the matching subschema separately, so
one bad value does not affect the others.

- The agent MAY apply, adjust, or ignore any hint not listed in `required`.
- When the agent does not honor a hint (an unlisted model, an invalid value, a value the client
  may not use, an option that is unavailable), it MUST NOT fail the request for that reason. It
  MUST proceed with its default, or another value it judges sensible, for that setting.
- A limit above the subschema's `maximum` SHOULD be clamped to the maximum. A limit below
  `minimum` MUST NOT be applied.
- The agent MAY explain a change in its normal response content, such as a text Part or the Task
  status message. This needs no structure from this extension.
- The agent MUST NOT apply a value outside its configuration schema.
- If the hint request is malformed (not an object, or `hints` missing or not an object), the agent
  ignores it and proceeds with its defaults.

### 5.2 Required Hints

The `required` list is an opt-in exception to Section 5.1.

- For each property in `required`, the agent MUST honor the hint exactly or reject the request
  with `InvalidParamsError`. Clamping a required limit counts as not honoring it.
- The rejection SHOULD include a `google.rpc.BadRequest` detail with one field violation per
  unmet hint, with `field` set to `metadata["<extension URI>"].hints.<property>`.
- If `required` is not an array of strings, or names a property absent from `hints`, the agent
  MUST reject the request with `InvalidParamsError`.
- `required` never overrides authorization.

Clients SHOULD keep `required` short.

## 6. Client-Supplied Tools and Agent Skills

`tools` and `agentSkills` give the agent capabilities chosen by the client. An agent accepts them
only if its configuration schema declares them, and the schema SHOULD constrain them (for example,
allowed URL patterns and `maxItems`).

| Property | Entry fields |
| :------- | :----------- |
| `tools` | `type` (v1 defines `"mcp"`, a remote MCP server over Streamable HTTP ([MCP tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools))), `name` (client label, unique in the array), `url` (`https`), `allowedTools` (optional array of tool names), `description` (optional). |
| `agentSkills` | `name` (the skill's `SKILL.md` name), `url` (`https`), `sha256` (hex digest of the bytes at `url`). Skills use the Agent Skills format ([Agent Skills](https://agentskills.io/specification)). |

The agent:

- MUST check each `url` against its own allowlist, and MUST refuse loopback, link-local, and
  private addresses unless the operator allows them, after DNS resolution and on every redirect.
- MUST NOT attach credentials it holds for other purposes to requests to client-supplied
  endpoints. It MAY use credentials obtained for that endpoint, presenting its own identity.
- MUST NOT start local processes from a client-supplied reference.
- MUST verify `sha256` before using a skill, and MUST NOT run scripts bundled in a skill outside a
  sandbox with no access to its credentials or to other Tasks' data.
- MUST NOT let a skill's `allowed-tools` field grant tools the agent would not otherwise expose.
- MUST make supplied tools and skills available only for the Task that supplied them, and SHOULD
  namespace tool names so they cannot shadow its own tools.
- MUST treat tool descriptions, tool results, and skill content as untrusted input that cannot
  change its security policy.

Clients MUST NOT put credentials in a hint request.

## 7. Examples

### 7.1 Request and Response

The card is the one in Section 3. The client asks for high effort, a model it is not permitted to
use, and an output limit above the agent's maximum.

```json
{
  "message": {
    "messageId": "9f1c2a7e-0b44-4d4e-9c1e-1d6c2f0e8a10",
    "role": "ROLE_USER",
    "parts": [{ "text": "Compare the 2024 and 2025 grid storage estimates." }]
  },
  "metadata": {
    "https://a2a-protocol.org/extensions/configuration-hints/v1": {
      "hints": { "reasoningEffort": "high", "model": "research-deep", "maxOutputTokens": 64000 }
    }
  }
}
```

The agent applies `high`, clamps the limit to 32000, and uses its default model. It says so in the
status message.

```json
{
  "task": {
    "id": "c6d0a8e4-4b1f-4a55-9a43-2f4b7a1d9e02",
    "contextId": "5b1e7c3a-1f2d-4c9e-8a6b-0d3e2f1a4b5c",
    "status": {
      "state": "TASK_STATE_COMPLETED",
      "message": {
        "messageId": "a41f0c2e-7d3b-4e6a-9b8c-5f2d1e0a3c47",
        "role": "ROLE_AGENT",
        "parts": [{ "text": "research-deep is not available for this account; used research-fast." }]
      }
    },
    "artifacts": [
      { "artifactId": "answer-1", "parts": [{ "text": "Estimates fell by roughly ..." }] }
    ]
  }
}
```

With `"required": ["model"]` in the hint request, the agent would instead return
`InvalidParamsError` before doing any work.

### 7.2 Client-Supplied MCP Tool

```json
{
  "hints": {
    "tools": [
      {
        "type": "mcp",
        "name": "acme-filings",
        "url": "https://tools.acme.example/mcp",
        "allowedTools": ["search_filings"]
      }
    ]
  }
}
```

## 8. Security Considerations

- **Resource exhaustion.** Clients can hint maximum effort and limits on every request. Agents
  MUST apply their own per-client quotas independent of hints.
- **Not authorization.** Hints and advertised values grant nothing. Authorization MUST be based on
  the authenticated client identity.
- **Disclosure.** The configuration schema reveals how the agent is built. Model identifiers can
  reveal vendor relationships. Advertising `model` is OPTIONAL; agents MAY use aliases or the
  extended Agent Card.
- **Prompt injection.** Supplied skills, tool descriptions, and tool results can carry
  instructions aimed at the agent's model (Section 6).
- **Server-side request forgery and confused deputy.** Client-supplied URLs direct the agent's
  network access. Section 6 requires allowlisting and forbids reuse of the agent's credentials.
- **Data exfiltration.** A supplied tool receives what the agent sends it. Agents SHOULD NOT send
  data from other principals or Tasks to it.
- **Free-text properties.** Agent-specific properties SHOULD use enumerated values. A free-text
  property that reaches the model gives clients prompt-level control.
