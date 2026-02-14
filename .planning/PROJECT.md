# Master Telegram Bot Assistant

## What This Is

A conversational AI assistant for Haven Home Loans CRM, accessible via Telegram. Loan officers can text naturally to check pipeline data, manage leads, scan emails, schedule follow-ups, and handle multi-step tasks — all without leaving their mobile messaging app. Uses Claude AI for natural language understanding with n8n workflow orchestration for reliable integrations.

## Core Value

**Instant mobile access to CRM data through natural conversation.** Loan officers can text "what's my pipeline?" or "who should I follow up with?" and get immediate, intelligent responses without opening the CRM web interface.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Maintain conversation context across messages (remember what we're discussing)
- [ ] Understand natural language queries (not just commands)
- [ ] Execute CRM queries (pipeline, leads, loans, partners, activities)
- [ ] Handle multi-step tasks with user approval (e.g., check calendar → draft email → send if approved)
- [ ] Learn user preferences over time (preferred follow-up times, notification settings)
- [ ] Provide contextual quick-action buttons after AI responses
- [ ] Route 80% of queries via fast rules, 20% via AI analysis (performance + cost optimization)
- [ ] Integrate with Outlook for email scanning and drafting
- [ ] Support voice-to-text input for mobile convenience
- [ ] Log all conversations for compliance and debugging

### Out of Scope

- **Complex business logic** — Assistant queries CRM APIs, doesn't replicate CRM functionality
- **Direct database writes** — All data modifications go through CRM API endpoints (audit trail)
- **Multi-user conversations** — 1:1 only (Harris → bot), not group chats
- **File uploads to CRM** — Can scan emails with attachments but not upload arbitrary files
- **Real-time notifications** — Proactive alerts (lock expiring, closing tomorrow) deferred to v2

## Context

**Background:**
- Haven Home Loans has a full-featured CRM (https://github.com/harriswat/havenhomeloans-mortgage-crm)
- CRM covers: Leads, Prospects (voice entry), Active Loans, Closed Loans, Referral Partners
- Owner (Harris) + 4 loan officers use the system
- Mobile-first workflow: Loan officers are often on the road meeting clients

**Why This Exists:**
- Loan officers need quick access to pipeline data without opening laptop
- Common queries: "What's my pipeline?", "Who closes this week?", "Any new leads?"
- Current friction: Must log into CRM web interface on mobile browser

**User Research:**
- Loan officers already use Telegram daily for team communication
- Voice-to-text highly requested (dictate notes while driving)
- Natural language preferred over memorizing commands
- Need multi-step workflows: "Check calendar and email Joe if I'm free tomorrow at 10"

**Technical Environment:**
- CRM: Next.js 15, PostgreSQL 17, Better Auth, Digital Ocean App Platform
- n8n: Already deployed ($49/mo), has Telegram bot configured (`@Bigterrys_bot`)
- Existing integrations: Outlook OAuth, Google Sheets, Deepgram (voice), Claude API

## Constraints

- **Tech Stack**: Must use n8n (already deployed), Claude API (already configured), PostgreSQL (existing CRM database), Redis (new - for sessions)
- **Budget**: Target $100-150/mo total (n8n $49, DB $15, Redis ~$10, Claude API ~$20-40)
- **Performance**: 80% of queries must respond <500ms (rule-based), 20% can be 2-3s (AI-powered)
- **Integration**: Must integrate via CRM APIs only (no direct database access from bot)
- **Security**: Role-based access (OWNER sees all data including commissions, LOs see only their data)
- **Mobile-First**: Primary interface is Telegram mobile app, design for thumb-friendly buttons

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Hybrid intent recognition (AI + rules) | 80/20 rule: most queries are simple ("pipeline") and can use fast rule matching; complex queries ("which VA loans are at risk?") need AI analysis | — Pending |
| Two-tier memory (Redis + PostgreSQL) | Redis for fast session access (<10ms), PostgreSQL for long-term learning and semantic search | — Pending |
| Tool calling vs pure prompting | Structured function calls (Claude's native tool use) prevent hallucination and ensure validated inputs | — Pending |
| n8n orchestration over custom code | Already deployed, visual debugging, no-code extensibility for future features | ✓ Good (confirmed by existing workflow success) |
| Separate repo from CRM | Telegram bot is a microservice that integrates via APIs, can be reused for other projects | — Pending |

---
*Last updated: 2026-02-14 after initialization*
