# Technology Stack

**Project:** Haven Home Loans Telegram AI Assistant
**Researched:** 2026-02-14

## Recommended Stack

### Conversational Interface

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Telegram Bot API | 9.3+ | User-facing chat interface | Already configured (@Bigterrys_bot). Mobile-first, free, supports rich media. Inline keyboards enable approval workflows. Single-user use case eliminates scaling concerns. |

### Orchestration Layer

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| n8n (self-hosted) | Latest (1.82+) | Workflow orchestration, tool routing, API integration | Already deployed on Digital Ocean ($49/mo). AI Agent node natively supports Claude via LangChain. Visual debugging, no-code extensibility, 1000+ integrations. All AI Agent nodes now default to Tools Agent pattern (simplified since v1.82). |

### Intelligence Layer

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Claude Haiku 4.5 | `claude-haiku-4-5-20251001` | Intent classification, tool calling, response generation (primary) | 4-5x faster than Sonnet at 1/3 cost ($1/$5 per MTok). Achieves ~90% of Sonnet's reasoning quality. Ideal for real-time chat where <3s latency matters. Supports tool use, parallel tool calls, and extended thinking. |
| Claude Sonnet 4.5 | `claude-sonnet-4-5-20250929` | Complex reasoning fallback, multi-step task planning | Use for 20% edge cases requiring deeper analysis (risk assessment, multi-step orchestration). $3/$15 per MTok. Better at asking clarifying questions vs. guessing. |

### Session Memory

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Valkey (DigitalOcean Managed) | 8.0 | Short-term conversation context, session state, pending approvals | Drop-in Redis replacement (100% compatible with Redis clients/libraries). DigitalOcean discontinued managed Redis (June 2025) and replaced with Valkey. $15/mo starting. 37% higher throughput and 30-60% lower latency than Redis 8.0. n8n Redis Chat Memory node works unchanged with Valkey. |

### Persistent Memory

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| PostgreSQL 17 (existing) | 17 | Long-term facts, user preferences, conversation summaries, audit logs | Already deployed and paid for ($15.15/mo). New `ai_memory` table fits naturally alongside existing CRM schema. Prisma ORM already configured. No additional infrastructure cost. |

### CRM API Layer

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Next.js API Routes | 15 | REST endpoints for assistant tool calls | Already the CRM framework. Add `/api/assistant/*` routes alongside existing routes. Server Actions already proven for CRUD. Consistent auth and data access patterns. |

### External Integrations

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Microsoft Graph API | v1.0 | Email scanning, calendar checks, draft emails | Already authenticated via OAuth. Free with Office 365 subscription. Covers Outlook email + calendar in one API. |
| Google Sheets API | v4 | Pipeline data bridge (interim) | Already integrated in existing workflows. Serves as fallback until CRM API endpoints are built. |

## Architecture Decision: n8n AI Agent Node vs. Raw HTTP Requests

**Decision: Use n8n's built-in AI Agent node for Claude integration, NOT raw HTTP Request nodes.**

**Rationale:**
- The AI Agent node handles the full tool-calling loop automatically (send tools, receive tool_use response, execute tool, return results, get final response)
- Built-in memory management via Redis Chat Memory sub-node
- Streaming support when paired with Chat Trigger or Webhook node
- Automatically handles parallel tool calls and sequential chains
- Less error-prone than manually implementing the Claude API conversation format in HTTP Request nodes

**Exception:** Use HTTP Request nodes for direct CRM API calls within tool execution workflows (the tools themselves), since those are simple request/response calls to your own endpoints.

**Confidence: HIGH** -- n8n docs explicitly recommend AI Agent node over manual HTTP for Claude integration.

## Architecture Decision: Haiku-First, Sonnet-Fallback

**Decision: Use Claude Haiku 4.5 as the primary model for all interactions. Escalate to Sonnet 4.5 only for explicitly complex requests.**

**Rationale:**
- Single-user bot (Harris only) means ~50-100 messages/day
- At Haiku pricing: ~$2-5/month for typical usage
- At Sonnet pricing: ~$8-15/month for same volume
- Haiku's <1s response time meets the <3s target for 90%+ queries
- Sonnet reserved for: multi-step task planning, risk analysis, ambiguous entity resolution

**Implementation:** n8n workflow routes to model based on complexity score or explicit escalation.

**Confidence: HIGH** -- Multiple benchmarks confirm Haiku 4.5 achieves 90% of Sonnet quality at 3x lower cost and 4-5x faster speed.

## Architecture Decision: Valkey over Redis

**Decision: Use DigitalOcean Managed Valkey instead of Redis.**

**Critical finding:** DigitalOcean discontinued Managed Caching (Redis) on June 30, 2025. All Redis clusters were automatically migrated to Valkey. Redis 7.2 reached EOL in February 2026.

**Why Valkey is fine:**
- Forked from Redis 7.2.4, 100% API-compatible
- Existing Redis clients, libraries, and n8n nodes work without modification
- Better performance: 37% higher throughput (SET), 60% lower latency (GET) vs Redis 8.0
- Backed by Linux Foundation, AWS, Google Cloud, Oracle
- BSD 3-clause license (no licensing risk)
- DigitalOcean manages failover, backups, encryption at rest + transit

**Risk:** Valkey and Redis are diverging over time. Redis-specific extensions (RediSearch, RedisJSON) may not be available. For this use case (session key-value storage with TTL), this divergence is irrelevant.

**Confidence: HIGH** -- Verified via DigitalOcean docs and multiple industry sources.

## Architecture Decision: Single Telegram Trigger Workflow

**Decision: Use ONE n8n workflow with a single Telegram Trigger, routing internally via Switch/IF nodes.**

**Critical constraint:** Telegram allows only one webhook per bot. Multiple workflows with Telegram triggers will overwrite each other -- only the last activated workflow receives messages.

**Implementation:**
1. Single "Command Router" workflow with Telegram Trigger
2. Switch node routes by message type (text, callback_query, voice)
3. Execute Workflow nodes call specialized sub-workflows
4. Sub-workflows return results to the router for Telegram reply

**Confidence: HIGH** -- Confirmed limitation in Telegram Bot API docs and n8n community.

## Architecture Decision: No pgvector for MVP

**Decision: Skip pgvector semantic search for persistent memory in Phase 1. Use simple key-value lookups and recency-based retrieval.**

**Rationale:**
- Single user with ~50-100 messages/day does not generate enough memory volume to benefit from vector similarity search
- DigitalOcean Managed PostgreSQL may not have pgvector extension available
- Key-value lookups with type/recency filters are sufficient for the use case
- Can add pgvector later if memory grows to hundreds/thousands of facts

**Confidence: MEDIUM** -- pgvector availability on DO Managed PostgreSQL not verified. Simple approach is pragmatic for single-user MVP.

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| Session store | Valkey (DO Managed) | Self-hosted Redis on Droplet | Managed service eliminates ops burden. Only $15/mo. DO no longer offers managed Redis. |
| Session store | Valkey (DO Managed) | Upstash Redis (serverless) | Another vendor to manage. DO Valkey keeps everything on one platform. Upstash is cheaper at very low volume but adds complexity. |
| LLM (primary) | Claude Haiku 4.5 | OpenAI GPT-4o-mini | Claude's tool calling is more reliable for structured CRM data. n8n has first-class Anthropic support. Haiku pricing is competitive. Already using Claude API key. |
| LLM (fallback) | Claude Sonnet 4.5 | Claude Opus 4.6 | Opus is overkill for a personal assistant. 5x Haiku cost. Sonnet provides sufficient reasoning quality for complex tasks. |
| Orchestration | n8n (existing) | Custom Node.js backend | n8n is already deployed and paid for. Visual debugging is invaluable. No additional infrastructure. Custom code only if n8n limitations arise. |
| Orchestration | n8n (existing) | LangChain/LangGraph standalone | Requires new deployment, more code to maintain. n8n already wraps LangChain internally. |
| Chat interface | Telegram | WhatsApp Business API | Telegram is free, already configured. WhatsApp Business API requires approval and Meta platform dependency. |
| Chat interface | Telegram | Custom web chat | Adds frontend dev work. Telegram is already installed, mobile-first, push notifications built in. |
| Streaming | Edit-based message streaming | sendMessageDraft | sendMessageDraft requires forum topic mode (not standard 1:1 DMs). Edit-based approach works in all chat types -- send initial message then editMessageText as tokens arrive. |

## Estimated Costs

### New Infrastructure

| Component | Monthly Cost | Status |
|-----------|-------------|--------|
| Valkey (DO Managed) | $15.00 | New |
| Claude API (Haiku primary) | $3-8 | New |
| Claude API (Sonnet fallback) | $2-5 | New |
| **New costs total** | **$20-28/mo** | |

### Existing Infrastructure (no change)

| Component | Monthly Cost | Status |
|-----------|-------------|--------|
| n8n (DO App Platform) | $49.00 | Existing |
| PostgreSQL (DO Managed) | $15.15 | Existing |
| Telegram API | $0.00 | Free |
| Microsoft Graph API | $0.00 | Included in Office 365 |

### Total

| | Monthly |
|---|---|
| Current infrastructure | $64.15 |
| + New (Valkey + Claude) | $20-28 |
| **Total** | **$84-92/mo** |

**ROI:** Saves 2-3 hours/week of CRM lookup time ($200-600/mo value at $25-50/hr). Positive ROI from day one.

## Installation / Setup

### Valkey (DigitalOcean)

```bash
# Create via DO Console or CLI
doctl databases create haven-valkey \
  --engine valkey \
  --region nyc3 \
  --size db-s-1vcpu-1gb \
  --num-nodes 1

# Get connection string
doctl databases connection haven-valkey

# Add to Doppler
doppler secrets set VALKEY_URL="valkeys://default:PASSWORD@HOST:PORT"
```

### Claude API

```bash
# Add API key to Doppler (already have one from existing integration)
doppler secrets set ANTHROPIC_API_KEY="sk-ant-..."

# n8n: Add Anthropic credential via console
# Settings > Credentials > New > Anthropic > paste API key
```

### n8n AI Agent Node Setup

```
1. Create new workflow: "Telegram AI Assistant - Command Router"
2. Add Telegram Trigger node (webhook mode, @Bigterrys_bot token)
3. Add AI Agent node
4. Connect sub-nodes:
   - Anthropic Chat Model (claude-haiku-4-5-20251001)
   - Redis Chat Memory (Valkey connection URL, TTL: 3600)
   - Tool nodes (HTTP Request Tool, Workflow Tool, etc.)
5. Configure system prompt with tool definitions
6. Add Telegram Send Message node for responses
```

### CRM API Endpoints

```bash
# Files to create in Next.js project
src/app/api/assistant/
  loans/active/route.ts      # GET - Pipeline overview
  loans/[id]/route.ts        # GET - Loan details
  loans/[id]/status/route.ts # PUT - Update status
  leads/recent/route.ts      # GET - Recent leads
  leads/[id]/contact/route.ts # POST - Log contact
  partners/top/route.ts      # GET - Top partners
  calendar/check/route.ts    # POST - Calendar availability
  email/scan/route.ts        # POST - Email summary
```

## Sources

### Verified (HIGH confidence)
- [Claude Tool Use Overview](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview) -- Official Anthropic docs
- [Claude Models Overview](https://platform.claude.com/docs/en/about-claude/models/overview) -- Pricing and model comparison
- [n8n AI Agent Node Docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/) -- Tools Agent documentation
- [n8n Anthropic Chat Model Node](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.lmchatanthropic/) -- Claude integration in n8n
- [n8n Redis Chat Memory Node](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.memoryredischat/) -- Session memory
- [DigitalOcean Managed Valkey](https://docs.digitalocean.com/products/databases/valkey/) -- Pricing, features, migration from Redis
- [DigitalOcean Managed Caching Discontinuation](https://www.digitalocean.com/blog/digitalocean-managed-caching) -- Redis to Valkey transition
- [Telegram Bot API](https://core.telegram.org/bots/api) -- Official API reference
- [Telegram Bot API Changelog](https://core.telegram.org/bots/api-changelog) -- sendMessageDraft, streaming features
- [n8n Telegram Trigger Common Issues](https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.telegramtrigger/common-issues/) -- Single webhook limitation

### Verified (MEDIUM confidence)
- [Valkey vs Redis Comparison (Better Stack)](https://betterstack.com/community/comparisons/redis-vs-valkey/) -- Compatibility analysis
- [Claude Haiku 4.5 Deep Dive (Caylent)](https://caylent.com/blog/claude-haiku-4-5-deep-dive-cost-capabilities-and-the-multi-agent-opportunity) -- Performance benchmarks
- [n8n AI Agent Guide (Strapi)](https://strapi.io/blog/build-ai-agents-n8n) -- 2026 patterns
- [Redis for GenAI Apps](https://redis.io/docs/latest/develop/get-started/redis-in-ai/) -- Session memory patterns
- [Redis Agent Memory Server](https://redis.github.io/agent-memory-server/) -- Two-tier memory architecture

### Background (LOW confidence -- inform design, verify before implementing)
- [Anthropic Advanced Tool Use Engineering Blog](https://www.anthropic.com/engineering/advanced-tool-use) -- Tool context management
- [n8n Community: AI Agent v3 Memory Pollution Issue](https://github.com/n8n-io/n8n/issues/22112) -- Known bug with Redis Chat Memory storing intermediate tool outputs
