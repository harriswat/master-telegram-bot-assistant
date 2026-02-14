# Architecture Patterns

**Domain:** Conversational AI assistant for mortgage CRM (Telegram + n8n + Claude + Valkey)
**Researched:** 2026-02-14
**Overall Confidence:** HIGH (verified across official docs, multiple production references, and existing codebase analysis)

---

## Recommended Architecture

### System Overview

The architecture is a **four-layer hybrid system** that routes 80% of queries through fast deterministic rules and 20% through Claude AI analysis. The layers are: Interface (Telegram), Orchestration (n8n), Intelligence (Claude API with model routing), and Memory (Valkey + PostgreSQL). This is not novel -- it is the dominant production pattern for conversational AI in 2026.

```
                     +------------------+
                     |    Telegram Bot   |
                     |  (Interface Layer) |
                     +--------+---------+
                              |
                              v
                     +------------------+
                     |  n8n Orchestrator |
                     | (Routing + State) |
                     +----+--------+----+
                          |        |
                +---------+        +----------+
                v                             v
   +-------------------+        +-------------------+
   |  Rule Engine       |        |  Claude AI Engine  |
   |  (80% fast path)   |        |  (20% flex path)   |
   |                    |        |                    |
   |  Pattern matching  |        |  Haiku: classify   |
   |  Direct API calls  |        |  + respond (most)  |
   |  Template responses|        |  Sonnet: complex   |
   |  <500ms            |        |  2-3s              |
   +--------+-----------+        +--------+-----------+
            |                             |
            +----------+------------------+
                       |
                       v
            +---------------------+
            |   Memory System     |
            |                     |
            |  Valkey (session)   |
            |  PostgreSQL (long)  |
            +---------------------+
                       |
                       v
            +---------------------+
            |   Backend APIs      |
            |                     |
            |  CRM (loans/leads)  |
            |  Microsoft Graph    |
            +---------------------+
```

### Component Boundaries

| Component | Responsibility | Communicates With |
|-----------|---------------|-------------------|
| Telegram Bot API | Message ingestion, response delivery, typing indicators, inline buttons, callback queries | n8n (webhook) |
| n8n Command Router | Receive messages, extract intent, route to rule engine or AI, manage state | Telegram, Rule Engine, Claude, Memory, Backend APIs |
| Rule Engine (n8n Switch node) | Fast pattern matching for known commands and common phrases | n8n Router (input), Backend APIs (output) |
| n8n AI Agent Node (Claude) | Intent classification, tool calling, natural language response generation | n8n Router (orchestrated by), Backend APIs (via tool calls) |
| Valkey Session Store | Short-term conversation context (last 10-20 messages), pending approvals, session state | n8n (via Redis Chat Memory node -- fully Valkey-compatible) |
| PostgreSQL Persistent Memory | Long-term facts, user preferences, task history, corrections | CRM API (via Prisma), n8n (via HTTP) |
| CRM API Layer (`/api/assistant/*`) | Loan pipeline, lead management, partner data, commission tracking | n8n Tool Executor (HTTP requests) |
| Microsoft Graph API | Email scanning, calendar checking, draft creation | n8n Tool Executor (HTTP requests) |

---

## Intent Recognition Architecture

### The Hybrid Router: Rules First, AI Second

This is the most critical architectural decision. The router must be a **two-stage cascade**, not a single LLM call.

**Stage 1: Fast Rule Engine (n8n Switch/Code node)**

Handles exact commands and high-frequency patterns. No LLM cost, no latency.

```
Rules (ordered by specificity):
  1. Slash commands:  /help, /pipeline, /email, /leads, /status
  2. Exact keywords:  "pipeline", "leads", "email", "calendar", "help"
  3. Phrase patterns:  "what's my pipeline", "check my email", "new leads"
  4. Greeting detect:  "hi", "hello", "hey" -> contextual greeting + menu
  5. Approval responses: "yes", "no", "approve", "cancel" (when pending approval exists)
```

Implementation in n8n: A Code node normalizes the message (lowercase, trim, remove punctuation), then a Switch node checks against rule patterns. If matched, route directly to the appropriate handler. If no match, pass to Stage 2.

**Stage 2: Claude AI Agent (Haiku 4.5 primary)**

Handles everything the rule engine cannot match. Use Claude Haiku 4.5 ($1/$5 per M tokens) as the primary model -- it achieves ~90% of Sonnet's quality at 3x lower cost and 4-5x faster speed.

**Key decision: Haiku for BOTH classification AND response generation.**

The prior architecture design proposed separate Haiku classification + Sonnet response generation steps. After deeper research, this two-LLM-call approach adds latency (two round trips) and complexity. Haiku 4.5 is capable enough for both tasks in a single call using the n8n AI Agent node with tool calling. Escalate to Sonnet 4.5 only when Haiku explicitly cannot handle the complexity (multi-step planning with 3+ conditional branches, risk analysis across many data points).

**Implementation via n8n AI Agent node:**
```
n8n AI Agent Node
  +-- Model: Anthropic Chat Model (claude-haiku-4-5-20251001)
  +-- Memory: Redis Chat Memory (Valkey connection)
  +-- Tools:
       +-- HTTP Request Tool (CRM API calls)
       +-- Workflow Tool (sub-workflow for calendar/email)
       +-- Code Tool (data formatting)
```

The AI Agent node handles the full loop internally: receives message + context, decides which tools to call, executes tools, generates natural response. No manual tool_use/tool_result parsing needed -- n8n's LangChain integration manages this.

### Why NOT Pure LLM Classification

Three reasons to keep the rule engine:
1. **Latency**: Rules execute in <50ms. Even Haiku adds 300-800ms.
2. **Cost**: Rules cost $0. At 100 messages/day, Haiku still adds $2-4/month.
3. **Reliability**: Rules never hallucinate. "pipeline" always means get_active_loans.

### Why NOT Pure Rules

Three reasons to keep the AI layer:
1. **Coverage**: Users will say "How's Kevin's loan looking?" -- no rule catches that.
2. **Entity extraction**: "Schedule a call with Emily for Thursday at 2pm" requires NLU.
3. **Context**: "How many are VA?" only makes sense in the context of a prior pipeline query.

---

## Two-Tier Memory System

### Architecture Decision: Valkey + PostgreSQL

**Critical update from research:** DigitalOcean discontinued Managed Caching (Redis) on June 30, 2025 and replaced it with Managed Valkey. Redis 7.2 reached EOL in February 2026. Valkey is a 100% compatible drop-in replacement (forked from Redis 7.2.4, BSD license, backed by Linux Foundation).

**Why two tiers:**
- Stuffing full conversation history into every Claude API call is expensive. A 20-message conversation is ~2000 tokens of context per call.
- PostgreSQL alone is too slow for session lookups on every message (5-20ms vs Valkey sub-1ms).
- Valkey alone lacks durability and structured querying for long-term knowledge.

### Tier 1: Session Memory (Valkey on DigitalOcean)

**What it stores:** The last 10-20 messages, current topic/entity focus, pending approval states, temporary multi-step task variables.

**Provider: DigitalOcean Managed Valkey** ($15/mo starting)

Rationale for DO Valkey over Upstash:
- Keeps all infrastructure on one platform (DO already runs n8n, PostgreSQL, and the CRM app)
- n8n's Redis Chat Memory node works unchanged with Valkey (same Redis protocol)
- No additional vendor relationship or billing
- Same VPC as n8n -- low-latency internal networking
- $15/mo is an acceptable cost for the simplicity and reliability of a managed service

**Key structure:**
```
Key:    telegram_session:{chat_id}
TTL:    3600 seconds (1 hour of inactivity)
Value:  JSON {
  messages: [
    { role: "user", content: "...", ts: "..." },
    { role: "assistant", content: "...", ts: "..." }
  ],
  context: {
    current_topic: "pipeline",
    mentioned_entities: ["Kevin Folsomtrexler"],
    last_tool_used: "get_active_loans",
    state: "IDLE"  // IDLE | AWAITING_APPROVAL | CLARIFYING
  },
  pending: {
    type: "email_approval",
    draft_id: "...",
    expires_at: "..."
  }
}
```

**Token management:** Keep only the last 10 messages (~1000 tokens). When exceeding 10 messages, summarize the oldest 5 into a compact context summary. This prevents context window bloat.

**Known issue (n8n AI Agent v3.0):** The AI Agent node saves full intermediate tool outputs into Redis Chat Memory, even when `returnIntermediateSteps = false` (GitHub issue #22112). Mitigation: Add a post-processing Code node that strips large tool outputs from the session before the save step.

### Tier 2: Persistent Memory (PostgreSQL)

**What it stores:** Learned facts, user corrections, preferences, compressed conversation summaries, task history.

**Schema (Prisma model):**
```prisma
model AiMemory {
  id          String     @id @default(cuid())
  userId      String     // Telegram chat_id
  memoryType  MemoryType
  key         String
  value       String     @db.Text
  metadata    Json?
  confidence  Float      @default(1.0)
  accessCount Int        @default(0)
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt
  expiresAt   DateTime?

  @@index([userId])
  @@index([memoryType])
  @@index([userId, memoryType])
  @@map("ai_memory")
}

enum MemoryType {
  FACT
  PREFERENCE
  TASK
  CORRECTION
  CONVERSATION_SUMMARY
}
```

**No pgvector for MVP.** For a single-user system with <1000 memories, simple key/type/recency queries are sufficient. Add pgvector when memory volume justifies it. DigitalOcean Managed PostgreSQL may not even have the pgvector extension available.

**Memory retrieval pattern (on each message):**
```
1. Load session from Valkey (sub-1ms)
2. Query persistent memory:
   - WHERE userId = {chat_id}
   - AND (key ILIKE '%{entity}%' OR memoryType = 'PREFERENCE')
   - ORDER BY accessCount DESC, updatedAt DESC
   - LIMIT 5
3. Inject both into Claude's system prompt as context
```

---

## Tool Calling Architecture

### Use n8n's AI Agent Node, Not Manual HTTP Request Tool Calling

**Key research finding:** n8n's AI Agent node (since v1.82.0) uses the Tools Agent pattern exclusively. It handles the full Claude tool-calling loop internally via LangChain:

1. Send message + tool definitions to Claude
2. Receive tool_use response
3. Execute the requested tool (via connected sub-nodes)
4. Return tool_result to Claude
5. Get final natural language response
6. Repeat if Claude requests additional tools

This eliminates the need to manually parse `tool_use` blocks, construct `tool_result` messages, or manage the multi-turn conversation format in HTTP Request nodes.

**Tool definition pattern:**

Tools are defined through n8n sub-nodes connected to the AI Agent node:

- **HTTP Request Tool**: For CRM API calls (GET /api/assistant/loans/active, etc.)
- **Workflow Tool**: For complex operations that are themselves n8n workflows (email scan, calendar check)
- **Code Tool**: For data formatting, calculations, template generation

Each tool node includes a name and description that Claude uses to decide when to invoke it.

**Best practice from Anthropic docs:** Use `strict: true` in tool definitions for guaranteed schema conformance. Provide extremely detailed descriptions with usage examples in the system prompt -- Claude's tool selection quality improves dramatically with contextual examples.

### Tool Execution via Sub-Workflows

For tools that require multiple steps (email scanning = Graph API call + Claude summarization), use n8n's "Workflow Tool" to call a separate sub-workflow:

```
Main Router Workflow
  +-- AI Agent Node
       +-- Tool: "get_active_loans" (HTTP Request to CRM API)
       +-- Tool: "scan_emails" (Workflow Tool -> Email Scan Sub-Workflow)
       +-- Tool: "check_calendar" (Workflow Tool -> Calendar Sub-Workflow)
```

Benefits of sub-workflow separation:
- Each sub-workflow is independently testable with mock input
- Keeps the main router workflow under 20-25 nodes
- Sub-workflow errors are isolated and do not crash the main router
- New capabilities = new sub-workflow + new tool node connection

---

## State Management in Conversational Flows

### State Machine for Conversation Modes

The bot operates in one of several modes at any time. The current mode determines how messages are interpreted.

```
States:
  IDLE          -> Default. Accept any message. Route through rule/AI engine.
  AWAITING_APPROVAL -> User must respond yes/no/edit to a pending action.
  CLARIFYING    -> Bot asked a clarifying question. Next message is the answer.

Transitions:
  IDLE + user_message                    -> process normally -> IDLE
  IDLE + AI_generates_approval_request   -> AWAITING_APPROVAL
  AWAITING_APPROVAL + "yes"/"approve"    -> execute action -> IDLE
  AWAITING_APPROVAL + "no"/"cancel"      -> abort -> IDLE
  AWAITING_APPROVAL + "edit"             -> request edits -> AWAITING_APPROVAL
  AWAITING_APPROVAL + timeout(15min)     -> auto-cancel -> IDLE
  IDLE + AI_asks_clarification           -> CLARIFYING
  CLARIFYING + user_response             -> resume with context -> IDLE
```

**Where state lives:** In Valkey session memory under `context.state`. The n8n workflow checks this field before routing messages. A "yes" in AWAITING_APPROVAL mode means "approve the draft email," not "yes I want to see my pipeline."

---

## CRM API Endpoint Architecture

### Route Structure

```
/api/assistant/
  loans/
    active/          GET  -- Active pipeline (filters: status, product, closing_soon, lock_expiring)
    [id]/            GET  -- Single loan details
    [id]/status/     PUT  -- Update loan status (with activity log)
    search/          GET  -- Search by borrower name (fuzzy match)
  leads/
    recent/          GET  -- Recent leads (filter: days, status)
    [id]/contact/    POST -- Log contact attempt
    [id]/assign/     PUT  -- Assign to loan officer
  partners/
    top/             GET  -- Partners by tier/performance
    [id]/            GET  -- Partner details + metrics
  analytics/
    pipeline/        GET  -- Pipeline value summary
    commission/      GET  -- Commission summary (OWNER only)
```

### Authentication

The bot's CRM API endpoints use a shared API key (stored in Doppler), NOT user session auth. The n8n workflow includes an `X-Assistant-Key` header. The API validates this key and operates with OWNER-level access (since Harris is the only user).

```typescript
// src/app/api/assistant/middleware.ts
export function validateAssistantKey(request: Request): boolean {
  const key = request.headers.get('X-Assistant-Key');
  return key === process.env.ASSISTANT_API_KEY;
}
```

---

## Error Handling and Fallback Strategies

### Tiered Error Recovery

**Tier 1: Automatic Retry (invisible to user)**
- API timeouts: Retry up to 3 times with exponential backoff (1s, 2s, 4s)
- CRM API errors: Retry once, then fall back to cached data or friendly message
- Claude API errors: Retry once, then fall back to rule-based response

**Tier 2: Graceful Degradation (transparent to user)**
- Claude unavailable: "I'm having a bit of trouble right now. Here's what I can tell you from the CRM directly: [rule-based response]"
- CRM API down: "I can't reach the CRM at the moment. Try again in a minute, or check directly at [CRM URL]"
- Valkey unavailable: Fall back to stateless processing. Log the failure.

**Tier 3: User-Facing Error (clear, actionable)**
- Unknown intent: "I'm not sure what you're asking. Try 'pipeline', 'leads', or 'email'. Or type 'help' for the full menu."
- Tool failure: "I tried to [action] but ran into an issue. Want me to try again?"
- Stale approval: "That action has expired. Want me to set it up again?"

### Logging for Debugging

Every message flow should be logged to an `ai_logs` table:
```json
{
  "timestamp": "...",
  "chat_id": "...",
  "user_message": "...",
  "route_taken": "rule|haiku|sonnet",
  "tools_called": [{"name": "...", "params": {}, "result_summary": "...", "duration_ms": 150}],
  "response_text": "...",
  "total_response_time_ms": 450,
  "errors": []
}
```

---

## Performance Targets

| Query Type | Path | Target | Token Cost |
|-----------|------|--------|------------|
| Exact command (/pipeline) | Rule engine only | <500ms | $0 |
| Common phrase ("what's my pipeline") | Rule engine only | <500ms | $0 |
| Ambiguous query ("How's Kevin doing?") | Haiku AI Agent | 1.5-3s | ~$0.001 |
| Complex reasoning ("Which loans are at risk?") | Sonnet AI Agent | 2-4s | ~$0.004 |
| Approval response ("yes") | State machine (no AI) | <300ms | $0 |
| Multi-step task | Haiku/Sonnet + multiple tools | 3-5s | ~$0.005 |

---

## Patterns to Follow

### Pattern 1: Typing Indicator for AI Operations

Send `sendChatAction("typing")` immediately when entering the AI path. Users tolerate 3-second waits when they see activity. Without it, they wonder if the bot is broken.

### Pattern 2: Progressive Disclosure for Complex Results

Send a brief summary first, then offer details via inline keyboard:
```
User: "pipeline"
Bot:  "You have 8 active loans ($3.7M). 2 closing this week, 1 lock expiring.
       [Closings] [Lock Warnings] [Full List]"
```

### Pattern 3: Entity Resolution with Options

When multiple database matches exist for a name, present options:
```
User: "How's Sarah doing?"
Bot:  "I found two Sarahs:
       1. Sarah Martinez - $525k purchase, closing Feb 17
       2. Sarah Thompson - $450k refi, in underwriting
       Which one?"
```

### Pattern 4: Approval Workflow with Inline Keyboards

Store pending actions in Valkey with a unique ID, present inline keyboard:
```
Bot: "I'll email Joe: 'Hi Joe, I'm free tomorrow at 10 AM...'
      [Approve] [Edit] [Cancel]"
Callback data: "approve:abc123", "edit:abc123", "cancel:abc123"
```
Always call `answerCallbackQuery` as the first response step (prevents 15-second loading spinner).

### Pattern 5: Single Entry Point with Sub-Workflow Routing

All messages enter through ONE workflow (Telegram webhook constraint), fan out via Execute Workflow nodes:
```
[Telegram Trigger] --> [Load Session] --> [Switch: Message Type]
  +-- text --> [Rule Check] --> [Match? Fast Path : AI Agent Path]
  +-- callback_query --> [Approval Handler]
  +-- voice --> [Transcribe] --> [text path]
--> [Save Session] --> [Send Response]
```

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Multiple Telegram Triggers Across Workflows
Telegram allows ONE webhook per bot. Only the last activated workflow receives messages. Use internal routing instead.

### Anti-Pattern 2: Storing Full Tool Output in Session Memory
Known n8n bug (GitHub #22112): AI Agent v3.0 saves full intermediate tool outputs into Redis Chat Memory. Add a post-processing sanitizer node.

### Anti-Pattern 3: Sending Raw JSON to Telegram
Always format as human-readable Markdown. Let Claude or templates handle formatting.

### Anti-Pattern 4: Using Claude Opus for Every Request
At ~50-100 messages/day, Opus costs 5x Haiku for marginal improvement. Haiku 4.5 for 95%+, Sonnet for complex cases. Never Opus.

### Anti-Pattern 5: Direct Database Access from Bot
The bot talks to `/api/assistant/*` endpoints, never the PostgreSQL database directly. This preserves auth, validation, audit logging, and schema independence.

---

## Scalability Considerations

| Concern | 1 User (now) | 5 Users (team) | 50+ Users |
|---------|-------------|----------------|-----------|
| Telegram | Single bot, single user | Single bot, whitelist chat_ids | Multiple bots or role-based routing |
| Valkey | $15/mo single node, <1MB | Same -- trivial for 1GB | Add HA cluster |
| Claude API | ~$5-8/month | ~$25-40/month | Aggressive caching, prompt optimization |
| CRM API | Negligible | Negligible | Add rate limiting |
| n8n executions | ~100/day | ~500/day | Monitor limits, consider dedicated instance |
| Memory | Simple key-value | Per-user isolation (chat_id scoped) | Add pgvector |

Current architecture supports 5 users with zero changes -- Valkey keys are already scoped by `chat_id`, CRM API is stateless.

---

## Sources

### Official Documentation (HIGH confidence)
- [Claude Tool Use Overview](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview) -- Anthropic official
- [Claude Tool Use Implementation](https://platform.claude.com/docs/en/agents-and-tools/tool-use/implement-tool-use) -- Anthropic official
- [n8n AI Agent Node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/) -- n8n official
- [n8n Tools AI Agent](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/tools-agent/) -- n8n official
- [n8n Redis Chat Memory](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.memoryredischat/) -- n8n official
- [DigitalOcean Managed Valkey](https://docs.digitalocean.com/products/databases/valkey/) -- DO official
- [Telegram Bot API](https://core.telegram.org/bots/api) -- Telegram official

### Architecture Patterns (MEDIUM confidence)
- [AI Agent Memory: Build Stateful AI Systems](https://redis.io/blog/ai-agent-memory-stateful-systems/) -- Redis blog
- [n8n AI Agent and Tool Execution](https://deepwiki.com/n8n-io/n8n/5.4-ai-agent-and-tool-execution) -- DeepWiki
- [Build AI Agents with n8n (2026)](https://strapi.io/blog/build-ai-agents-n8n) -- Strapi
- [Multi-Agent Solutions in n8n](https://hatchworks.com/blog/ai-agents/multi-agent-solutions-in-n8n/) -- HatchWorks
- [Valkey vs Redis (Better Stack)](https://betterstack.com/community/comparisons/redis-vs-valkey/) -- Better Stack

### Known Issues
- [n8n GitHub #22112: AI Agent stores full tool outputs in Redis memory](https://github.com/n8n-io/n8n/issues/22112)
- [n8n Telegram Trigger: Single webhook limitation](https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.telegramtrigger/common-issues/)
