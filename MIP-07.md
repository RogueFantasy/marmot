# MIP-07

## AI Inference Pool

**draft** · **optional**

*This document defines an AI inference pool for Marmot groups. A designated executor forwards member prompts to an AI provider and relays results back to the group. Members gain shared access to models while individual authorship remains obscured.*

-----

## Overview

AI providers generally reserve frontier models for KYC'd, paying users. For users, their prompt history becomes a map of their thinking. This MIP specifies how Marmot groups can function as inference pools to reassign that attribution to other group members, broaden model access, and share cost:

1. **Anonymity** — designed to limit the provider's ability to attribute prompts to individual members
1. **Access** — members without provider credentials use the pool
1. **Cost** — inference cost is shared across the group or subsidized by the executor

## Design Scope

This protocol exists for anyone who treats the "prompt history as cognitive fingerprint" threat as a priority. It moves prompt attribution from the AI provider to a trusted group member. The executor is fully trusted with prompt content and authorship. The improvement is that members choose the executor, whose actions are visible to the entire group. Against the provider, the protocol provides attribution obfuscation — the provider sees prompts but cannot directly attribute them to individual authors. See [Security Considerations](#security-considerations) for a detailed analysis of this trust tradeoff.

### Privacy Boundary

- **External:** Prompts are attributed to the executor.
- **Internal:** Prompt content and authorship are visible to the group.

## Pool Message Types

All pool messages are MLS Application Messages within Group Events (kind: 445) per MIP-03. **CRITICAL:** Per MIP-03 security requirements, all inner events MUST remain unsigned (no `sig` field) and MUST NOT include `h` tags or other group identifiers.

This MIP uses event kinds 9706, 9707, and 9708. These kinds exist exclusively within MLS-encrypted group events and are never published directly to relays. Implementations SHOULD ignore unrecognized kinds in the 9700–9799 range gracefully.

### Pool Configuration (kind: 9706)

The admin sets the pool's baseline configuration — model selection, executor designation, and pool parameters. Authors MAY customize individual prompts. The executor has final discretion over what reaches the provider.

```json
{
  "kind": 9706,
  "pubkey": "<admin_nostr_pubkey>",
  "created_at": 1693876500,
  "content": "",
  "tags": [
    ["pool_version", "mip07-v1"],
    ["model", "<required_model_identifier>"],
    ["executor", "<designated_executor_pubkey>"],
    ["executor_backup", "<optional_backup_executor_pubkey>"],
    ["normalize", "rewrite"],
    ["system", "<optional_default_system_prompt>"],
    ["rotation_period", "<seconds>"],
    ["rotation_order", "<pubkey_1>", "<pubkey_2>", "<pubkey_3>"],
    ["params", "<optional_default_json_params>"]
  ]
}
```

#### Field Specifications

| Tag | Description | Required |
|---|---|---|
| `pool_version` | Protocol version. Current version: `"mip07-v1"`. | Yes |
| `model` | The model that prompts are expected to be executed against under the current configuration. | Yes |
| `executor` | Designated executor's Nostr pubkey for the current configuration. Must have access to the specified model. | Yes |
| `executor_backup` | Backup executor's Nostr pubkey. If the primary executor is unresponsive, the backup MAY execute pending prompts. Implementations determine the appropriate delay before the backup acts. | No |
| `normalize` | Minimum stylometric normalization level. Values: `"none"`, `"basic"`, `"rewrite"`. See [Stylometric Normalization](#stylometric-normalization). | Yes |
| `system` | Default system prompt included with all inference requests. Authors MAY override per-prompt. | No |
| `rotation_period` | Duration in seconds for each executor's duty period. String-encoded integer. Omit when rotation is not configured. | No |
| `rotation_order` | Ordered list of member pubkeys for executor duty rotation. Omit when rotation is not configured. | No |
| `params` | JSON-encoded default parameters passed to the provider (e.g., `temperature`, `top_p`). Per-prompt `params` override these defaults. | No |

#### Configuration Authority

**CRITICAL:** Only group admins (as defined in `admin_pubkeys` per MIP-01) MAY publish Pool Configuration messages. Clients MUST verify admin status before processing configuration updates. The most recent valid configuration (by `created_at` timestamp) takes precedence.

#### Configuration Transitions

When rotation is configured, the admin cycles through `rotation_order`, publishing a new config for each transition. When a new configuration is published, the outgoing executor is expected to complete or reject pending prompts. The incoming executor picks up any that remain unresolved — duplicate discard by `pool_id` prevents duplicate results. Unavailable executors are skipped based on rejection messages, unanswered prompts, or out-of-band communication.

### Prompt Submission (kind: 9707)

A member submits a prompt to the pool.

```json
{
  "kind": 9707,
  "pubkey": "<author_nostr_pubkey>",
  "created_at": 1693876800,
  "content": "",
  "tags": [
    ["pool_id", "<random_32_byte_hex>"],
    ["conversation_id", "<optional_random_32_byte_hex>"],
    ["prompt", "<prompt_text>"],
    ["system", "<optional_override>"],
    ["params", "<optional_override_json>"]
  ]
}
```

#### Field Specifications

| Tag | Description | Required |
|---|---|---|
| `pool_id` | Unique identifier for this prompt. MUST be 32 cryptographically random bytes, hex-encoded (64 characters). | Yes |
| `conversation_id` | Random 32-byte hex ID reused by clients across turns in a multi-turn conversation. | No |
| `prompt` | The prompt text. MUST be UTF-8 encoded. | Yes |
| `system` | Override the config's default system prompt for this prompt. | No |
| `params` | JSON-encoded parameters that override the config's defaults for this prompt. | No |

### Prompt Result (kind: 9708)

The designated executor returns a result.

```json
{
  "kind": 9708,
  "pubkey": "<executor_nostr_pubkey>",
  "created_at": 1693877100,
  "content": "",
  "tags": [
    ["pool_id", "<original_pool_id>"],
    ["response", "<ai_response_text>"],
    ["status", "complete"],
    ["prompt_count", "<executor_prompt_count_this_config>"],
    ["prompt_length", "<character_count_post_normalization>"],
    ["usage", "<optional_json_usage>"],
    ["reason", "<reason_code_if_rejected_or_error>"],
    ["detail", "<optional_human_readable_explanation>"]
  ]
}
```

#### Field Specifications

| Tag | Description | Required |
|---|---|---|
| `pool_id` | The `pool_id` from the original Prompt Submission. | Yes |
| `response` | The AI provider's response text. MUST be UTF-8 encoded. Present when `status` is `"complete"` or `"error"`. | Conditional |
| `status` | `"complete"` for success. `"error"` if the provider returned an error (`response` contains the error message). `"rejected"` if the executor declined to run the prompt. | Yes |
| `prompt_count` | Running count of prompts the executor has processed under the current configuration. String-encoded integer. Present when `status` is `"complete"`. | Conditional |
| `prompt_length` | Character count of the prompt as submitted to the provider (post-normalization). String-encoded integer. Present when `status` is `"complete"`. | Conditional |
| `usage` | JSON-encoded token usage: `{"prompt_tokens": 150, "completion_tokens": 500}`. Present when `status` is `"complete"` and token counts are available. | Conditional |
| `reason` | Machine-readable reason code for rejections or errors. Values: `tos_violation`, `rate_limited`, `model_unavailable`, `prompt_too_long`, `quota_exhausted`, `other`. Present when `status` is `"rejected"` or `"error"`. | Conditional |
| `detail` | Human-readable explanation providing additional context. Present when `status` is `"rejected"` or `"error"`. | No |

Rejection is always preferable to silent drops. If the executor cannot run a prompt for any reason, they should publish a result with `status: "rejected"` rather than ignoring the prompt.

## Executor Behavior

The designated executor watches the group channel for kind: 9707 messages and processes them as follows:

1. **Duplicate check:** Skip if a kind: 9708 already exists for this `pool_id`
1. **Normalize:** Apply stylometric normalization at or above the `normalize` level in the current configuration. See [Stylometric Normalization](#stylometric-normalization).
1. **Execute:** Send the prompt to the AI provider using the model and parameters from the current configuration, with any per-prompt overrides the executor chooses to apply
1. **Respond:** Publish a kind: 9708 result to the group — with the appropriate `status` and `reason` if execution failed, or with the response if successful

## Security Considerations

### Threat Model

**Protects against:**

- **Device exposure:** Non-executing members never interact with AI providers directly — their devices carry no provider credentials and are not exposed to provider-side security risks.
- **AI provider profiling:** The provider sees the executor, not the author. Across a group of N members, the provider cannot directly determine which member authored any given prompt.
- **Prompt history correlation:** Isolating any individual member's history from the executor's account requires additional analysis (stylometric, temporal, or topical) rather than simple attribution.

**Does not protect against:**

- **Malicious actors:** The executor can log, modify, or selectively drop prompts before forwarding them to the provider. Conversely, a prompt author can craft prompts designed to extract information from the executor. Both parties accept these risks by participating in the pool.
- **Stylometric analysis (partially mitigated):** Prompt writing style may correlate authors across prompts. See [Stylometric Normalization](#stylometric-normalization).
- **Timing analysis:** Submission timing patterns may reduce anonymity. See [Anonymity Set](#anonymity-set).

Every prompt and its result are visible to the entire group. Executor-facing obligations throughout this specification are enforced by this transparency, not by protocol mechanics. See executor activity tracking under [Implementation Requirements](#implementation-requirements).

### Mitigations

#### Stylometric Normalization

Prompt writing style can fingerprint authors even when identity metadata is stripped. Normalization removes identifying signal from individual prompts and makes traffic from multiple authors appear consistent, reinforcing the appearance of a single user on the executor's account. Authors MAY also normalize prompts locally before submission, ensuring neither the executor nor the provider sees the original writing style.

The executor MAY apply a stronger normalization level than the config specifies. The normalization hierarchy from weakest to strongest is: **none → basic → rewrite**.

**Level: `none`**

No normalization. Prompts are forwarded to the provider as submitted.

**Level: `basic`**

Rule-based normalization of punctuation, whitespace, formatting, and stylistic markers applied by the executor before forwarding. Implementations SHOULD document their exact normalization rules and share them with the group.

**Level: `rewrite`**

The executor runs all prompts through a separate model before forwarding to the target provider. This model rewrites the prompt in neutral language while aiming to preserve semantic intent, technical specifics, code blocks, and structured data.

#### Anonymity Set

The anonymity set is bounded by the number of active group members. Groups with more active members offer more timing privacy, but numbers alone do not guarantee strong anonymity. A group of 50 where one member submits 80% of prompts has a weak anonymity set despite its size.

There is also tension between anonymity set size and operational feasibility — more active members improve anonymity but require more trust, increase executor load and exposure, and broaden topical diversity.

### Operational Considerations

#### Group Composition

Groups function best as inference pools when members share a rough domain of activity. Topical coherence strengthens the anonymity set by making mixed traffic from the executor's account plausible as a single user's behavior. Groups spanning unrelated domains may create detectable traffic patterns on the executor's account — the provider may not identify individual authors, but anomalous topic diversity could flag the account.

#### Usage Accountability

The executor determines how members access the model — API key, subscription, or local instance. Payment arrangements depend on this choice. Payments between members and the executor can create financial attribution trails that undermine the pool's anonymity properties.

## Implementation Requirements

**Clients MUST:**

- Support sending and receiving kinds 9706, 9707, and 9708
- Verify admin status for kind: 9706 per MIP-01
- Verify MLS sender matches inner event pubkey per MIP-03
- Ensure all inner events remain unsigned and contain no `h` tags per MIP-03
- Display the number of members who have submitted prompts to the pool
- Track per-member usage (prompt counts, prompt lengths, and token usage) by matching results to their original authors
- Discard duplicate results for the same `pool_id` (keep first by `created_at`)

**Clients SHOULD:**

- Display pool status (pending prompt age, current model, normalization level)
- Surface per-member usage summaries to the group
- Warn users when their submission volume may reduce anonymity
- Track executor activity (response times, rejection rates, unanswered prompts)
- Support encrypted file attachments using MIP-04 media encryption

## Versioning

Version `mip07-v1`. Communicated via the `pool_version` tag in Pool Configuration. Implementations SHOULD ignore unknown tags gracefully for forward compatibility.

## References

- MIP-01: Group Construction & Marmot Group Data Extension
- MIP-03: Group Messages
- MIP-04: Encrypted Media
- Marmot Threat Model
- [RFC 9420: Messaging Layer Security](https://www.rfc-editor.org/rfc/rfc9420)
