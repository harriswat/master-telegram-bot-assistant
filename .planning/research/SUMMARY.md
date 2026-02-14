# Research Summary: Telegram AI Executive Assistant

**Domain:** Conversational AI / Telegram Bot for Mortgage CRM
**Researched:** 2026-02-14
**Overall confidence:** HIGH

## Executive Summary

Building a conversational AI assistant that balances speed, cost, and intelligence requires a layered hybrid architecture -- not a monolithic LLM-for-everything approach. The 2026 production consensus across Rasa, Voiceflow, Redis, Anthropic, and n8n documentation is clear: use deterministic rules for predictable queries and LLMs only where natural language understanding is genuinely needed. Hybrid systems benchmarked by Voiceflow outperform pure LLM solutions at 3-5x lower cost.

The project's existing infrastructure (n8n on DigitalOcean, PostgreSQL, Telegram bot, Claude API) aligns well with the recommended architecture. The primary new components are: (1) a DigitalOcean Managed Valkey session store for conversation memory, (2) a persistent memory table in the existing PostgreSQL database, (3) CRM API endpoints for the bot to consume, and (4) n8n AI Agent workflows with Claude Haiku tool calling.

**Key architectural refinement from this research:** Use Claude Haiku 4.5 as a single-model solution for BOTH intent classification AND response generation, not a two-model pipeline (Haiku classify + Sonnet respond). The n8n AI Agent node handles the full tool-calling loop internally via LangChain. Sonnet 4.5 is reserved only for explicitly complex multi-step reasoning tasks (~5% of queries). This simplifies the architecture and cuts latency by eliminating one LLM round trip.

**Critical infrastructure finding:** DigitalOcean discontinued Managed Redis (June 2025) and replaced it with Managed Valkey. Valkey is a 100% API-compatible Redis fork (from Redis 7.2.4) backed by the Linux Foundation. It works with all existing Redis clients, libraries, and n8n's Redis Chat Memory node without modification. Performance is actually better: 37% higher throughput and 30-60% lower latency than Redis 8.0.

## Key Findings

**Stack:** n8n orchestration + Claude Haiku 4.5 (primary) / Sonnet 4.5 (fallback) + DigitalOcean Managed Valkey (session) + PostgreSQL (persistent memory + CRM data) + Telegram Bot API

**Architecture:** Four-layer hybrid system (Interface, Orchestration, Intelligence, Memory) with 80/20 rule/AI split and single Telegram Trigger workflow routing

**Critical pitfalls:**
- Telegram allows ONE webhook per bot -- must use single workflow with internal routing
- n8n AI Agent v3.0 has known memory pollution bug (GitHub #22112) -- full tool outputs stored in Redis
- DO no longer offers managed Redis -- use Valkey instead

## Implications for Roadmap

Based on research, suggested phase structure:

1. **Infrastructure Setup (Valkey + CRM API Endpoints)** - Foundation that everything else depends on
   - Addresses: Session memory, data access layer
   - Avoids: Stateless interactions, direct database access from bot

2. **Core Query Assistant (Rule Engine + AI Agent)** - Handle 100% of read queries
   - Addresses: Pipeline queries, lead lookups, partner stats, closing schedule
   - Avoids: Building write operations before read operations work reliably

3. **Write Operations + Approval Workflows** - Modify CRM data via bot
   - Addresses: Status updates, lead assignment, contact logging, inline keyboards
   - Avoids: Unconfirmed write operations (always require approval)

4. **Proactive Alerts + Scheduled Tasks** - Bot reaches out to user
   - Addresses: Lock expiration alerts, closing date reminders, morning briefing
   - Avoids: Building scheduled features before real-time features are stable

5. **External Integrations (Email + Calendar)** - Microsoft Graph integration
   - Addresses: Email scanning, calendar checks, draft emails, meeting scheduling
   - Avoids: Complex OAuth flows before core CRM functionality is proven

6. **Learning + Refinement** - Persistent memory, rule pattern expansion
   - Addresses: Long-term memory, user preferences, usage-based optimization
   - Avoids: Over-engineering memory before observing real usage patterns

**Phase ordering rationale:**
- Valkey and CRM API endpoints must come first because both the rule engine and AI engine depend on them
- Read operations before write operations because reads are safe and validate the data layer
- Approval workflows before external integrations because the approval pattern (inline keyboards) is needed for email/calendar actions
- Alerts and scheduling after core functionality because they compose simpler capabilities
- Learning/refinement last because it requires real usage data to be meaningful

**Research flags for phases:**
- Phase 1: Verify DigitalOcean Managed Valkey networking with n8n App Platform (same VPC? connection string format?)
- Phase 3: n8n inline keyboard callback handling may need separate Telegram Trigger configuration (callback_query vs message)
- Phase 5: Microsoft Graph OAuth token refresh in n8n for long-running bot -- needs validation

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All components verified against official docs, current pricing, and DO availability |
| Features | HIGH | Feature landscape derived from existing CRM capabilities and mortgage domain |
| Architecture | HIGH | Hybrid rule/AI pattern verified across Rasa, Voiceflow, Redis, Anthropic, n8n docs |
| Pitfalls | HIGH | Error patterns verified from production post-mortems and community discussions |
| Cost model | MEDIUM | Based on usage estimates (50-100 msgs/day). Actual costs depend on real patterns |
| n8n memory bug | HIGH | Confirmed via GitHub issue #22112. Workaround documented but may need testing |
| Valkey compatibility | HIGH | Verified drop-in replacement. Same Redis protocol. DO documentation confirms |

## Gaps to Address

- **DO Valkey + n8n networking**: Verify n8n App Platform can reach DO Managed Valkey (same VPC or public access needed?)
- **n8n AI Agent + Anthropic prompt caching**: Need to test whether n8n's built-in Anthropic Chat Model node supports prompt caching headers, or if custom HTTP Request is needed
- **Telegram callback_query in n8n**: The Telegram Trigger node handles both messages and callback queries, but routing between them needs implementation testing
- **pgvector on DO Managed PostgreSQL**: Not needed for MVP, but if future phases require it, verify extension availability
- **Claude Haiku tool calling quality**: Haiku 4.5 is recommended for tool calling, but tool selection accuracy should be monitored vs. Sonnet during early usage
