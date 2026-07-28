# Memory Retention Policy Guide

This guide explains how to use the memory retention feature in OpenSearch ML Commons to automatically manage the lifecycle of agentic memory. It is written for both human operators and AI agents that interact with the memory container APIs.

---

## Table of Contents

1. [What Is Memory Retention?](#what-is-memory-retention)
2. [Memory Types at a Glance](#memory-types-at-a-glance)
3. [Quick Start](#quick-start)
4. [Retention Policy Structure](#retention-policy-structure)
5. [Setting a Retention Policy on a Container](#setting-a-retention-policy-on-a-container)
6. [Updating a Retention Policy](#updating-a-retention-policy)
7. [Opting Out of Retention](#opting-out-of-retention)
8. [Pinning Memories](#pinning-memories)
9. [How the Retention Job Works](#how-the-retention-job-works)
10. [Feature Flags and Admin Controls](#feature-flags-and-admin-controls)
11. [Cluster-Level Settings (Admin)](#cluster-level-settings-admin)
12. [Validation Rules and Error Messages](#validation-rules-and-error-messages)
13. [Worked Examples](#worked-examples)
14. [FAQ](#faq)

---

## What Is Memory Retention?

When AI agents use memory containers to store conversations (sessions), distilled knowledge (long-term memory), and audit trails (history), that data grows without bound. Without lifecycle management:

- Storage costs increase continuously.
- Agents retrieve stale or contradictory memories, degrading response quality.
- Larger context windows drive up inference costs.

The **memory retention policy** solves this by letting you define rules that automatically delete old or excess memories on a schedule. You set the rules; a background job enforces them.

> **Retention is opt-in.** The feature is gated by a master switch, `plugins.ml_commons.memory.retention_enabled`, which **defaults to `false`**. Until an administrator enables it, the memory container APIs reject any `retention_policy` or `pinned` input with a 403, and the background job short-circuits without deleting anything. See [Feature Flags and Admin Controls](#feature-flags-and-admin-controls) to turn it on.

---

## Memory Types at a Glance

A memory container holds four types of memory. Retention rules apply to three of them:

| Memory Type | What It Stores | Supports `retention_days` | Supports `max_count` | Supports `pinned` |
|---|---|---|---|---|
| **sessions** | Conversation sessions between a user and an agent | Yes | Yes | Yes |
| **long-term** | Distilled knowledge extracted from conversations | Yes | Yes | Yes |
| **history** | Immutable audit trail of all interactions | No | Yes | No |
| **working** | Individual messages within a session | Not directly configurable | Not directly configurable | No |

**Working memory** cannot have its own retention rule. It is always deleted when its parent session expires. To control how long messages live, configure retention on sessions.

---

## Quick Start

First, an administrator enables retention (it is off by default):

```json
PUT /_cluster/settings
{
  "persistent": {
    "plugins.ml_commons.memory.retention_enabled": true
  }
}
```

Then create a memory container with a retention policy that keeps sessions for 30 days (max 100), long-term memories capped at 5000, and history capped at 50,000:

```json
POST /_plugins/_ml/memory_containers/_create
{
  "name": "my-agent-memory",
  "configuration": {
    "embedding_model_type": "TEXT_EMBEDDING",
    "embedding_model_id": "your-embedding-model-id",
    "embedding_dimension": 1024,
    "llm_id": "your-llm-model-id",
    "disable_session": false,
    "disable_history": false,
    "strategies": [
      {
        "type": "SEMANTIC",
        "namespace": ["user_id"]
      }
    ],
    "retention_policy": {
      "sessions": {
        "retention_days": 30,
        "max_count": 100
      },
      "long-term": {
        "max_count": 5000
      },
      "history": {
        "max_count": 50000
      }
    }
  }
}
```

> `disable_session` and `disable_history` are both shown as `false` for clarity, but that is their default — you can omit them and sessions and history are still enabled. They are included here to make explicit that all three memory types (`sessions`, `long-term`, and `history`) are active and their retention rules meaningful.

That's it. The background job (runs every 24 hours by default) will enforce these rules automatically.

---

## Retention Policy Structure

A retention policy is a JSON object nested inside `configuration.retention_policy` on the memory container. It contains up to three keys, one per eligible memory type:

```json
{
  "configuration": {
    "retention_policy": {
      "sessions": {
        "retention_days": <positive integer or null>,
        "max_count": <positive integer or null>
      },
      "long-term": {
        "retention_days": <positive integer or null>,
        "max_count": <positive integer or null>
      },
      "history": {
        "max_count": <positive integer or null>
      }
    }
  }
}
```

### Field Reference

| Field | Type | Unit | Description |
|---|---|---|---|
| `retention_days` | Integer or null | Days | Delete memories older than this many days. Age is measured from the memory's `last_updated_time`. |
| `max_count` | Integer or null | Count | Keep at most this many memories. When the count is exceeded, the oldest are deleted first. Sessions and long-term memory are ordered by `last_updated_time`; history is ordered by `created_time`. |

**Both fields are optional and independent.** You can set one, both, or neither for each memory type. When both are set, they operate as an OR condition: a memory is deleted if it violates either rule.

### Constraints

- Both `retention_days` and `max_count` must be positive integers (greater than zero) when provided.
- `retention_days` is not supported on the `history` type. The API returns a 400 error if you try.
- The `working` key is not allowed. The API returns a 400 error with guidance to configure sessions instead.
- You only need to include the memory types you want to manage. Omitted types have no retention enforcement.

---

## Setting a Retention Policy on a Container

### At creation time

Include `retention_policy` in the create request:

> Assumes a container configured with an LLM (`llm_id`) and strategies — see the [Quick Start](#quick-start) example above. Without them, the container never stores long-term or history memory, so those rules have nothing to act on.

```json
POST /_plugins/_ml/memory_containers/_create
{
  "name": "customer-support-agent",
  "configuration": {
    "retention_policy": {
      "sessions": {
        "retention_days": 60,
        "max_count": 500
      },
      "long-term": {
        "max_count": 5000
      }
    }
  }
}
```

### If you do not provide a policy

When no `retention_policy` is specified on creation, **no retention enforcement occurs by default.** Nothing is automatically deleted.

The only exception: if retention is enabled (`retention_enabled=true`) **and** your cluster administrator has explicitly configured default retention settings (see [Cluster-Level Settings](#cluster-level-settings-admin)), those values are applied to the container at creation time. Both conditions are required — while retention is disabled, no policy is ever stamped. These admin defaults are completely optional and are all disabled out of the box. If your admin has not set them up, a container without an explicit policy simply has no retention — data grows without limit until you add a policy yourself.

---

## Updating a Retention Policy

Use a PUT request to modify the policy on an existing container:

```json
PUT /_plugins/_ml/memory_containers/{memory_container_id}
{
  "configuration": {
    "retention_policy": {
      "sessions": {
        "max_count": 200
      }
    }
  }
}
```

### Merge behavior

Updates use **field-level merge**, which means:

- **Memory types you include** are merged into the existing policy. Fields you specify are updated; fields you omit within that type are unchanged.
- **Memory types you omit** are left untouched.
- **To remove a single field**, send it explicitly as `null`:
  ```json
  { "configuration": { "retention_policy": { "sessions": { "retention_days": null } } } }
  ```
  This removes `retention_days` from sessions while preserving `max_count`.
- **To remove an entire memory type's rule**, send its value as `null`:
  ```json
  { "configuration": { "retention_policy": { "long-term": null } } }
  ```

### Examples of merge behavior

Starting policy:
```json
{
  "sessions": { "retention_days": 30, "max_count": 100 },
  "long-term": { "max_count": 5000 }
}
```

| Update request | Resulting policy |
|---|---|
| `{"sessions": {"max_count": 50}}` | sessions: days=30, count=50; long-term: count=5000 |
| `{"sessions": {"retention_days": null}}` | sessions: count=100 (days removed); long-term: count=5000 |
| `{"history": {"max_count": 10000}}` | sessions: days=30, count=100; long-term: count=5000; history: count=10000 |

---

## Opting Out of Retention

To disable all retention enforcement for a container, set the policy to `null`:

```json
PUT /_plugins/_ml/memory_containers/{memory_container_id}
{
  "configuration": {
    "retention_policy": null
  }
}
```

This is an explicit opt-out. The retention job will skip this container entirely, even if cluster-level defaults are configured. This is distinct from simply not having a policy (which allows defaults to be backfilled).

To opt back in later, provide a new concrete policy in a subsequent update.

---

## Pinning Memories

Pinning a memory exempts it from all retention enforcement. The retention job never deletes a pinned memory, regardless of its age or the current count.

### Pin a session

Pinning a session preserves the entire conversation, including all of its working memory messages.

```json
PUT /_plugins/_ml/memory_containers/{id}/memories/sessions/{session_id}
{
  "pinned": true
}
```

### Pin a long-term memory

```json
PUT /_plugins/_ml/memory_containers/{id}/memories/long-term/{memory_id}
{
  "pinned": true
}
```

### Unpin a memory

Set `pinned` to `false`:

```json
PUT /_plugins/_ml/memory_containers/{id}/memories/sessions/{session_id}
{
  "pinned": false
}
```

> **Note:** `pinned` is sent as a top-level field in the update body, not wrapped in an `update_content` object. Setting `pinned` requires `retention_enabled` to be `true` — otherwise the update is rejected with a 403 (the flag is dead metadata when the job is not running).

### Pinning rules

- **Sessions**: can be pinned. Protects the session and all its working memory from deletion.
- **Long-term**: can be pinned. Protects that specific memory from deletion.
- **Working memory**: cannot be pinned. Pin the parent session instead.
- **History**: cannot be pinned.
- **Pinned memories do not count toward `max_count`.** If you have `max_count: 100` and 120 sessions where 30 are pinned, the job sees 90 non-pinned sessions and keeps the newest 100 non-pinned ones (no deletions in this case).
- **Pinning does not reset the memory's age.** Only content changes (adding messages to a session, updating memory content) extend lifetime by bumping `last_updated_time`.

---

## How the Retention Job Works

A background job runs on a schedule (default: every 24 hours) and processes all memory containers. Understanding what it does helps you predict its behavior.

### Execution order

For each container with a retention policy, the job executes these phases in order:

1. **Session retention** (time-based, then count-based)
2. **Long-term memory retention** (time-based, then count-based)
3. **History retention** (count-based only)
4. **Working memory TTL** (only for session-disabled containers)
5. **Orphan sweep** (runs after all containers are processed)

### Session retention in detail

When `retention_days` is set: sessions whose `last_updated_time` is older than the threshold are deleted. Active conversations (recently messaged) are safe because adding messages bumps `last_updated_time`.

When `max_count` is set: if the number of non-pinned sessions exceeds the cap, the oldest sessions (by `last_updated_time`) are removed until the count is within bounds.

**Cascade behavior:** When a session is deleted, all of its working memory messages are deleted first, then the session document itself. Conversations are never left with gaps.

### Long-term memory retention in detail

Works identically to sessions: time-based deletion on `last_updated_time`, then count-based on `last_updated_time`, oldest first. Pinned memories are excluded from both.

> **Note:** On very large backlogs, each count-based (`max_count`) pass evicts at most 50,000 documents per type per run (the oldest first). A larger backlog converges over successive runs.

### History retention in detail

Count-based only. Oldest entries (by `created_time`) are deleted when the non-pinned count exceeds `max_count`.

### Working memory TTL

Only applies to containers with `disable_session: true` (session-less containers). In these containers, working memory has no parent session to cascade from, so a cluster-level TTL (`plugins.ml_commons.memory.working_memory_ttl_days`) governs when orphaned messages are cleaned up. This TTL **defaults to `-1` (disabled)** — sessionless working memory is kept indefinitely; the pass short-circuits whenever the value is `<= 0`. An admin must set a positive value (1–365 days) to age it out.

### Orphan sweep

Identifies working-memory documents whose parent session no longer exists (for example, if a session was manually deleted) and removes them. This prevents accumulation of unreachable data. Also removes completely unattributable working-memory documents older than `plugins.ml_commons.memory.orphan_ttl_days` (default 7 days).

The sweep has two safeguards against wiping legitimate data:

- **First-observation grace period.** The first time the sweep sees a container, it stamps a baseline timestamp (write-once) and deletes nothing. It defers all orphan deletion for that container until `baseline + orphan_ttl_days` has elapsed. This gives pre-existing working memory a full window to acquire a backing session before it can be swept.
- **Lazy session creation.** When a client adds working memory under its own `session_id` without first calling create-session, the add-memory path idempotently creates a minimal backing session document. The sweep then sees a live session and does not treat that working memory as orphaned. (The system deliberately does **not** backfill sessions for old pre-existing data, since on old data it cannot tell "never created a session" apart from "the user deleted the session.")

Distinct `session_id` enumeration is capped at 50,000 per run; larger sets converge over multiple runs.

### Staleness window

Because the job runs periodically (not in real time), there is a window between when a memory becomes eligible for deletion and when it is actually removed. This window is at most one job interval (default 24 hours). Expired memories may still appear in queries during this window.

---

## Feature Flags and Admin Controls

Memory retention is governed by two independent cluster settings that act as on/off switches at different levels. Understanding which one does what prevents confusion when things aren't behaving as expected.

### The two switches

```
┌─────────────────────────────────────────────────────────────────────────┐
│  plugins.ml_commons.agentic_memory_enabled          (default: true)     │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  Controls: ALL agentic memory APIs                                │  │
│  │  (create/update/get/delete containers and memories)               │  │
│  │                                                                   │  │
│  │  If false: every memory API returns 403 Forbidden.                │  │
│  │  The retention job is never even registered.                      │  │
│  │  Nothing memory-related works at all.                             │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  plugins.ml_commons.memory.retention_enabled        (default: FALSE)    │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  Master opt-in switch for the retention FEATURE.                  │  │
│  │  Controls: the background retention job AND acceptance of         │  │
│  │  retention_policy / pinned input on the container APIs.           │  │
│  │                                                                   │  │
│  │  Default false: retention is OFF. Any request carrying a          │  │
│  │  retention_policy or a pinned field is rejected with 403.         │  │
│  │  The job short-circuits and deletes nothing.                      │  │
│  │  Other memory APIs (create containers, add messages, get,         │  │
│  │  search) still work normally.                                     │  │
│  │                                                                   │  │
│  │  Set true to enable: policies and pins are accepted, and the      │  │
│  │  job enforces them on schedule.                                   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### What each combination means

| `agentic_memory_enabled` | `retention_enabled` | What happens |
|---|---|---|
| `true` | `false` | **This is the default.** All memory APIs work normally — create containers, add messages, get, search. But `retention_policy` and `pinned` input is rejected with 403, and the job deletes nothing. |
| `true` | `true` | Retention feature is on. Policies and pins are accepted; the job enforces them on schedule. |
| `false` | (irrelevant) | All memory APIs return 403. The retention job never registers. The entire agentic memory feature is off. |

### Do I need to change either of these?

**`agentic_memory_enabled`** defaults to `true` — the memory APIs work out of the box. **`retention_enabled`** defaults to **`false`** — the retention feature is opt-in, so you must explicitly enable it before you can set any policy or pin any memory. You need to change these if:

- **You want to use retention at all** — an admin must set `retention_enabled: true` first. Until then, create/update requests carrying a `retention_policy` (or a `pinned` field on a memory) return 403.
- **You are an admin who wants to pause retention enforcement** after enabling it — for example, you suspect the job is deleting something it shouldn't, or you're doing a migration and want to freeze all data in place temporarily. Set `retention_enabled` back to `false`.
- **Your organization does not use agentic memory** and wants to disable the feature entirely — set `agentic_memory_enabled: false`.

### How to pause retention (emergency stop)

If something is being deleted that shouldn't be, immediately run:

```json
PUT /_cluster/settings
{
  "persistent": {
    "plugins.ml_commons.memory.retention_enabled": false
  }
}
```

This takes effect immediately (dynamic setting — no restart needed). The next time the job fires, it will log a message and exit without touching any data. All your container policies remain saved; they just stop being enforced.

### How to resume retention

```json
PUT /_cluster/settings
{
  "persistent": {
    "plugins.ml_commons.memory.retention_enabled": true
  }
}
```

On the next scheduled run (within one job interval), the job will resume enforcing all policies as normal.

### How to disable agentic memory entirely

If your cluster does not use memory containers at all and you want to turn off the feature:

```json
PUT /_cluster/settings
{
  "persistent": {
    "plugins.ml_commons.agentic_memory_enabled": false
  }
}
```

After this, any API call to `/_plugins/_ml/memory_containers/...` returns:

```json
{
  "error": {
    "type": "status_exception",
    "reason": "The Agentic Memory APIs are not enabled. To enable, please update the setting plugins.ml_commons.agentic_memory_enabled"
  },
  "status": 403
}
```

### Common mistakes

| Mistake | What actually happens | Fix |
|---|---|---|
| Setting `retention_enabled: false` and expecting policies to still be enforced | No deletions occur anywhere. The setting is a global pause, not per-container. | Use `"retention_policy": null` on specific containers to opt them out individually, or leave the global setting at `true`. |
| Setting `agentic_memory_enabled: false` thinking it only disables retention | All memory APIs break with 403. Agents can no longer read or write memories. | Use `retention_enabled: false` instead — it only stops deletions. |
| Changing `retention_enabled` and expecting immediate deletions | The job runs on a schedule (default every 24 hours). Changes take effect on the next run. | Reduce `retention_job_interval_hours` to 1 for faster enforcement, or wait for the next cycle. |

---

## Cluster-Level Settings (Admin)

Cluster administrators can configure retention behavior using dynamic cluster settings. All settings use the prefix `plugins.ml_commons.memory.` and can be updated at runtime without a restart.

### Master switch

| Setting | Default | Description |
|---|---|---|
| `retention_enabled` | `false` | Master opt-in switch for the retention feature. When `false` (the default), the container APIs reject `retention_policy`/`pinned` input with 403 and the job deletes nothing. Set to `true` to enable retention cluster-wide. |

### Job schedule and throttling

| Setting | Default | Range | Description |
|---|---|---|---|
| `retention_job_interval_hours` | 24 | 1 - 168 | How often the retention job runs, in hours. |
| `retention_job_throttle_seconds` | 5 | 1 - 60 | Pause between containers during job execution to reduce cluster load. |

### Cleanup TTLs

| Setting | Default | Range | Description |
|---|---|---|---|
| `working_memory_ttl_days` | -1 (off) | -1 - 365 | TTL for working memory in session-disabled containers. Defaults to `-1` (disabled) — sessionless working memory is kept indefinitely unless an admin sets a value > 0. |
| `orphan_ttl_days` | 7 | 1 - 365 | TTL for unattributable orphaned working-memory documents. |

### Cluster-level default retention policy (optional, admin-configured)

These settings are **all disabled by default (`-1`)**. Out of the box, no retention policy is applied to any container unless the user explicitly provides one at creation time. The retention job does nothing to containers that have no policy.

An administrator can optionally configure these settings to establish organization-wide baseline retention rules. If retention is enabled (`retention_enabled=true`) and any of these settings are set to a value greater than zero, containers that (a) have no explicit policy and (b) have not explicitly opted out will receive a policy built from these values — either at creation time or on the next job run via backfill. While `retention_enabled` is `false`, defaults are never stamped.

**If you do not set these, nothing happens automatically. There is no built-in retention behavior without explicit configuration.**

| Setting | Default | Range | Effect when set > 0 |
|---|---|---|---|
| `default_session_retention_days` | -1 (off) | -1 - 3650 | Applies `retention_days` to sessions on new containers |
| `default_session_max_count` | -1 (off) | -1 - 1,000,000 | Applies `max_count` to sessions on new containers |
| `default_long_term_max_count` | -1 (off) | -1 - 1,000,000 | Applies `max_count` to long-term on new containers |
| `default_history_max_count` | -1 (off) | -1 - 10,000,000 | Applies `max_count` to history on new containers |

### Example: setting cluster defaults (optional)

If your organization wants all containers to have a baseline retention policy without requiring every user to set one manually, an admin can configure these.

> **The numbers below are illustrative only.** There are no built-in or recommended default values — every setting ships as `-1` (off). Choose values that fit your own storage budget and compliance requirements; do not treat these figures as guidance.

```json
PUT /_cluster/settings
{
  "persistent": {
    "plugins.ml_commons.memory.default_session_retention_days": 90,
    "plugins.ml_commons.memory.default_session_max_count": 5000,
    "plugins.ml_commons.memory.default_long_term_max_count": 2000,
    "plugins.ml_commons.memory.default_history_max_count": 100000
  }
}
```

Once set:
- Newly created containers that do not specify their own policy will inherit these values at creation time.
- Existing containers that have no policy and have not opted out will receive them on the next job run via backfill.
- Containers that already have an explicit policy (or explicitly opted out with `"retention_policy": null`) are never affected.

**Important:** Defaults are baked into the container at the time they are applied. If you change cluster defaults later, previously created containers are not retroactively updated. To change a specific container's policy, use the update API.

### Removing cluster defaults

To go back to "no automatic policy," reset the settings to `-1`:

```json
PUT /_cluster/settings
{
  "persistent": {
    "plugins.ml_commons.memory.default_session_retention_days": -1,
    "plugins.ml_commons.memory.default_session_max_count": -1,
    "plugins.ml_commons.memory.default_long_term_max_count": -1,
    "plugins.ml_commons.memory.default_history_max_count": -1
  }
}
```

After this, no new containers will receive automatic policies, and the backfill will stop applying to existing containers without policies.

---

## Validation Rules and Error Messages

| Condition | HTTP Status | Error Message |
|---|---|---|
| `retention_days` set to 0 or negative | 400 | `retention_days must be a positive integer or null` |
| `max_count` set to 0 or negative | 400 | `max_count must be a positive integer or null` |
| `retention_days` specified for history | 400 | `retention_days is not supported for history memory type` |
| `working` key included in the policy | 400 | `Working memory retention cannot be configured directly. Working memory is deleted when its parent session expires. To control message lifetime, configure retention on "sessions" instead.` |
| Unrecognized memory type key | 400 | `unknown memory type: <key>` |
| `pinned` set when adding working memory | 400 | `pinned field is not supported for working memory type. To preserve a conversation, pin the session instead.` |
| `retention_policy` supplied on create/update while `retention_enabled = false` | 403 | `Cannot set retention_policy: the memory retention feature is not enabled. To enable it, please update the cluster setting plugins.ml_commons.memory.retention_enabled` |
| `pinned` field present in a memory request while `retention_enabled = false` | 403 | `Cannot set pinned: the memory retention feature is not enabled. To enable it, please update the cluster setting plugins.ml_commons.memory.retention_enabled` |

> An explicit `"retention_policy": null` (opt-out) is still accepted while the feature is disabled, since clearing retention is consistent with the feature being off.

---

## Worked Examples

### Example 1: Customer support agent with aggressive cleanup

A high-volume support agent that processes thousands of conversations daily. You want to keep only recent sessions and a moderate knowledge base.

> Assumes a container configured with an LLM (`llm_id`) and strategies — see the [Quick Start](#quick-start) example above. Without them, the container never stores long-term or history memory, so those rules have nothing to act on.

```json
POST /_plugins/_ml/memory_containers/_create
{
  "name": "high-volume-support-agent",
  "configuration": {
    "retention_policy": {
      "sessions": {
        "retention_days": 7,
        "max_count": 1000
      },
      "long-term": {
        "retention_days": 180,
        "max_count": 10000
      },
      "history": {
        "max_count": 100000
      }
    }
  }
}
```

**What happens:** Sessions older than 7 days are deleted. If more than 1000 non-pinned sessions exist before 7 days have passed, the oldest are evicted early. Long-term memories are kept for 6 months or until 10,000 accumulate. History keeps the most recent 100,000 entries.

### Example 2: Research assistant with long memory

A knowledge-heavy agent where long-term memory is critical and conversations are secondary.

> Assumes a container configured with an LLM (`llm_id`) and strategies — see the [Quick Start](#quick-start) example above. Without them, the container never stores long-term or history memory, so those rules have nothing to act on.

```json
POST /_plugins/_ml/memory_containers/_create
{
  "name": "research-assistant",
  "configuration": {
    "retention_policy": {
      "sessions": {
        "retention_days": 14,
        "max_count": 200
      },
      "long-term": {
        "max_count": 50000
      }
    }
  }
}
```

**What happens:** Sessions expire after 2 weeks. Long-term memory grows up to 50,000 entries with no time-based expiry (knowledge is only evicted when the count is exceeded, oldest first). History has no policy, so no history cleanup occurs.

### Example 3: Protecting important conversations

Pin a session that contains an important troubleshooting thread so it is never deleted, regardless of retention rules:

```json
PUT /_plugins/_ml/memory_containers/{id}/memories/sessions/{session_id}
{
  "pinned": true
}
```

The pinned session and all its messages will persist indefinitely. It does not count against `max_count`, so it will not block other sessions from being kept.

### Example 4: Understanding OR logic between retention_days and max_count

Container has: `sessions: { retention_days: 30, max_count: 100 }`

Scenario A: You have 50 sessions. One is 31 days old.
- Result: The 31-day-old session is deleted (violates `retention_days`). The other 49 remain.

Scenario B: You have 110 sessions, all less than 30 days old.
- Result: The 10 oldest sessions are deleted (violates `max_count`). 100 remain.

Scenario C: You have 80 sessions, all less than 30 days old.
- Result: No deletions. Neither rule is violated.

### Example 5: Disabling just time-based retention

You want count-based limits only, with no time-based expiry:

```json
{
  "configuration": {
    "retention_policy": {
      "sessions": {
        "retention_days": null,
        "max_count": 500
      },
      "long-term": {
        "retention_days": null,
        "max_count": 10000
      }
    }
  }
}
```

### Example 6: Completely opting out

```json
PUT /_plugins/_ml/memory_containers/{memory_container_id}
{
  "configuration": {
    "retention_policy": null
  }
}
```

No retention enforcement of any kind will apply to this container, even if cluster defaults are configured.

---

## FAQ

### Does the retention job delete data immediately when a rule is violated?

No. The job runs on a schedule (default every 24 hours). There is a staleness window of up to one job interval where expired memories may still be visible in query results.

### What happens to working memory when a session is deleted?

All working memory (messages) belonging to that session are deleted first, then the session itself is removed. Conversations are never left in a partial state.

### Can I set retention rules on working memory directly?

No. Working memory lifecycle is tied to its parent session. Configure `sessions` retention to control how long messages live.

### Does pinning a session reset its age?

No. Pinning is a metadata operation and does not change `last_updated_time`. Only adding messages to a session extends its lifetime. However, this does not matter for retention because pinned sessions are never deleted regardless of age.

### What if I have more pinned sessions than max_count?

The job will never delete pinned items. It logs a warning that the container is growing beyond the cap due to pins, but enforcement only applies to non-pinned items. To bring the container back under control, review and unpin sessions that no longer need protection.

### Do cluster default changes affect existing containers?

No. Defaults are applied to a container once (at creation time or on first backfill). Changing cluster defaults later does not update containers that already have a policy. Use the update API to modify individual containers.

### What is the difference between "no policy" and "opted out"?

- **No policy** (field absent): The container may receive cluster defaults via backfill on the next job run.
- **Opted out** (`"retention_policy": null`): The container is permanently skipped by the retention job. Defaults are never backfilled. This is an active choice.

### Can I trigger the retention job on demand?

Not in this version. The job runs on its configured interval. You can reduce the interval to 1 hour for faster enforcement if needed:

```json
PUT /_cluster/settings
{
  "persistent": {
    "plugins.ml_commons.memory.retention_job_interval_hours": 1
  }
}
```

### What OpenSearch version is required?

Memory retention policies require OpenSearch 3.8.0 or later.

### Does this work with multi-tenancy?

Not in this version. The retention job is disabled when multi-tenancy is active. Multi-tenant support is planned for a future release.

### My memory APIs are returning 403. What's wrong?

There are two distinct causes:

- **Every memory API returns 403** (including create container, add message, get, search): `plugins.ml_commons.agentic_memory_enabled` is `false`. Set it back to `true` to restore API access.
- **Only requests carrying `retention_policy` or `pinned` return 403**, while other memory APIs work: `plugins.ml_commons.memory.retention_enabled` is `false` (its default). This is expected — retention is opt-in. Enable it before setting policies or pinning memories. (An explicit `"retention_policy": null` is still accepted while disabled.)

Check both settings:

```json
GET /_cluster/settings?include_defaults=true&filter_path=*.plugins.ml_commons.agentic_memory_enabled,*.plugins.ml_commons.memory.retention_enabled
```

### I set a retention policy but nothing is being deleted. What should I check?

Walk through this checklist in order:

1. **Is `plugins.ml_commons.memory.retention_enabled` set to `true`?** It defaults to `false`. If it is `false`, the job no-ops — and you would not have been able to set a policy in the first place (the API returns 403). Enable it first.
2. **Is `plugins.ml_commons.agentic_memory_enabled` set to `true`?** If `false`, the job was never registered.
3. **Has enough time passed?** The job runs every `retention_job_interval_hours` (default 24 hours). Your policy won't be enforced until the next run.
4. **Is your container's policy actually set?** Check with `GET /_plugins/_ml/memory_containers/{id}` and inspect `configuration.retention_policy`.
5. **Are the memories pinned?** Pinned memories are exempt from all retention rules.
6. **Is multi-tenancy enabled?** The retention job is disabled when multi-tenancy is active.

---

## API Reference Summary

| Operation | Method | Endpoint | Body |
|---|---|---|---|
| Create container with policy | POST | `/_plugins/_ml/memory_containers/_create` | `{"configuration": {"retention_policy": {...}}}` |
| Update container policy | PUT | `/_plugins/_ml/memory_containers/{id}` | `{"configuration": {"retention_policy": {...}}}` |
| Opt out of retention | PUT | `/_plugins/_ml/memory_containers/{id}` | `{"configuration": {"retention_policy": null}}` |
| Pin a session | PUT | `/_plugins/_ml/memory_containers/{id}/memories/sessions/{session_id}` | `{"pinned": true}` |
| Unpin a session | PUT | `/_plugins/_ml/memory_containers/{id}/memories/sessions/{session_id}` | `{"pinned": false}` |
| Pin a long-term memory | PUT | `/_plugins/_ml/memory_containers/{id}/memories/long-term/{memory_id}` | `{"pinned": true}` |
| Enable retention (admin) | PUT | `/_cluster/settings` | `{"persistent": {"plugins.ml_commons.memory.retention_enabled": true}}` |
| Set cluster defaults | PUT | `/_cluster/settings` | `{"persistent": {"plugins.ml_commons.memory.default_session_retention_days": 90, ...}}` |
| Disable retention cluster-wide | PUT | `/_cluster/settings` | `{"persistent": {"plugins.ml_commons.memory.retention_enabled": false}}` |
