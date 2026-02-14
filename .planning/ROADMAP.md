# Roadmap: Master Telegram Bot Assistant

## Overview

This roadmap delivers a conversational AI assistant for Haven Home Loans CRM, accessible via Telegram. The build starts with infrastructure (Valkey session store, CRM API endpoints, authentication) and progressively layers on intelligence (hybrid rule/AI routing, Claude tool calling), write operations (status updates, approvals), proactive capabilities (alerts, morning briefing), external integrations (Outlook email and calendar), and finally voice input with persistent learning. Each phase delivers a complete, usable capability -- Harris can start querying his pipeline after Phase 2 and never needs to wait for all 6 phases.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Infrastructure & Foundation** - Valkey, CRM API endpoints, authentication, audit logging
- [ ] **Phase 2: Core Query Assistant** - Hybrid routing, Claude AI Agent, session memory, read queries
- [ ] **Phase 3: Write Operations & Approvals** - Status updates, lead assignment, inline keyboard approval workflows
- [ ] **Phase 4: Proactive Alerts & Scheduling** - Morning briefing, lock expiration alerts, closing reminders
- [ ] **Phase 5: Outlook Integration** - Email scanning, draft composition, calendar checks via Microsoft Graph
- [ ] **Phase 6: Voice Input & Persistent Learning** - Voice-to-text transcription, long-term memory, preference learning

## Phase Details

### Phase 1: Infrastructure & Foundation
**Goal**: All foundational services are running and the bot can receive messages, authenticate the user, and access CRM data through dedicated API endpoints
**Depends on**: Nothing (first phase)
**Requirements**: CRM-01 (partial -- API layer), LOG-01
**Success Criteria** (what must be TRUE):
  1. DigitalOcean Managed Valkey instance is provisioned, connection string stored in Doppler, and n8n can connect to it
  2. CRM has at least 5 read-only `/api/assistant/*` endpoints returning real data (active loans, loan details, loan search, recent leads, top partners)
  3. A single n8n workflow with Telegram Trigger receives messages from @Bigterrys_bot and replies with an echo (proving webhook works)
  4. Messages from non-whitelisted chat IDs are rejected silently and logged
  5. Every incoming message is logged to a `bot_logs` table with timestamp, chat_id, message text, and processing metadata
**Plans**: TBD

Plans:
- [ ] 01-01: Provision Valkey and configure CRM API endpoints
- [ ] 01-02: Build Telegram command router workflow with auth and logging

---

### Phase 2: Core Query Assistant
**Goal**: Harris can text the bot naturally and get accurate, formatted CRM data with contextual action buttons -- the bot remembers conversation context within a session
**Depends on**: Phase 1
**Requirements**: CONV-01, NLU-01, CRM-01, ROUTE-01, UX-01
**Success Criteria** (what must be TRUE):
  1. Harris can type "pipeline" or "what's my pipeline?" and receive a formatted summary of active loans with dollar amounts, statuses, and count -- in under 500ms for rule-matched queries
  2. Harris can ask "How's Kevin doing?" and the bot resolves the name to the correct loan record, showing status, amount, and key dates -- with disambiguation buttons if multiple matches exist
  3. Harris can ask "Any new leads?" and get a list of recent leads with names, sources, and assignment status
  4. After asking about pipeline, Harris can follow up with "How many are VA?" and the bot correctly interprets the pronoun reference using session context from Valkey
  5. Every bot response includes 2-4 contextual inline keyboard buttons suggesting logical next actions (e.g., after pipeline: [Closings This Week] [Lock Alerts] [New Leads])
**Plans**: TBD

Plans:
- [ ] 02-01: Build rule engine and hybrid intent router
- [ ] 02-02: Configure Claude AI Agent with tool calling and session memory
- [ ] 02-03: Implement response formatting, inline keyboards, and error handling

---

### Phase 3: Write Operations & Approvals
**Goal**: Harris can modify CRM data through the bot with explicit confirmation before any change is executed
**Depends on**: Phase 2
**Requirements**: TASK-01
**Success Criteria** (what must be TRUE):
  1. Harris can say "Move Kevin's loan to underwriting" and the bot presents the proposed change with [Confirm] [Cancel] buttons -- the status updates only after confirmation
  2. Harris can say "Assign the new lead to Brenden" and the bot identifies the lead, shows the assignment details, and waits for approval before executing
  3. Harris can say "Log a call with Sarah Martinez" and the bot creates a contact activity with timestamp and optional notes
  4. Pending approvals expire after 15 minutes with a friendly notification, and the bot correctly handles new messages while an approval is pending (does not confuse "yes" as approval for a different topic)
**Plans**: TBD

Plans:
- [ ] 03-01: Build write API endpoints and approval state machine
- [ ] 03-02: Implement inline keyboard callback handlers and approval workflows

---

### Phase 4: Proactive Alerts & Scheduling
**Goal**: The bot proactively delivers time-sensitive information without being asked -- becoming a daily habit rather than an on-demand tool
**Depends on**: Phase 2
**Requirements**: (Enriches CRM-01 -- no new requirement; this phase composes existing read capabilities into scheduled delivery)
**Success Criteria** (what must be TRUE):
  1. Harris receives a morning briefing at 7:30 AM every weekday summarizing: active pipeline count and value, today's closings, expiring rate locks, new leads since yesterday, and overdue follow-ups
  2. Harris receives a push notification when a rate lock is expiring within 3 days, with a [View Loan] [Draft Reminder] action button
  3. Harris receives a push notification when a closing is within 2 days, with a [View Details] button
  4. Alerts are not duplicated (each alert fires once per trigger condition, tracked via a sent-alerts log)
**Plans**: TBD

Plans:
- [ ] 04-01: Build morning briefing workflow and alert scheduling
- [ ] 04-02: Implement lock expiration and closing date alert triggers

---

### Phase 5: Outlook Integration
**Goal**: Harris can scan emails, draft replies, and check calendar availability through the bot -- with approval required before any email is sent
**Depends on**: Phase 3 (needs approval workflow for email send confirmation)
**Requirements**: EMAIL-01
**Success Criteria** (what must be TRUE):
  1. Harris can say "Check my email" and receive an AI-summarized overview of recent emails grouped by priority, with sender names and subject lines
  2. Harris can say "Draft a reply to Sarah about the closing" and the bot generates a contextually appropriate email draft, presenting it with [Send] [Edit] [Cancel] buttons
  3. Harris can say "Am I free tomorrow at 2?" and the bot checks his Outlook calendar and responds with availability
  4. No email is ever sent without explicit [Send] approval -- the bot always drafts first and waits for confirmation
**Plans**: TBD

Plans:
- [ ] 05-01: Build email scanning and AI summarization workflow
- [ ] 05-02: Build email drafting, calendar checks, and approval-gated sending

---

### Phase 6: Voice Input & Persistent Learning
**Goal**: Harris can send voice messages that the bot transcribes and acts on, and the bot remembers facts and preferences across sessions permanently
**Depends on**: Phase 2 (voice feeds into existing text pipeline), Phase 3 (voice commands may trigger write operations)
**Requirements**: VOICE-01, LEARN-01
**Success Criteria** (what must be TRUE):
  1. Harris can send a voice message and the bot transcribes it, shows the transcription, and processes the intent exactly as if it were typed text
  2. When Harris says via voice "Update Kevin's loan to processing at 500k and 6.75 rate", the bot extracts the structured data, presents it for confirmation, and executes on approval
  3. The bot remembers facts across sessions (e.g., "Harris prefers morning follow-ups", "Kevin's loan is the $500k refi") and uses them to provide more relevant responses
  4. Harris can correct the bot ("No, I meant the other Kevin") and the correction is stored permanently, improving future interactions
**Plans**: TBD

Plans:
- [ ] 06-01: Build voice message transcription pipeline (Deepgram integration)
- [ ] 06-02: Build persistent memory system (PostgreSQL ai_memory table + retrieval)

## Requirement Coverage

| Requirement | Description | Phase | Coverage Notes |
|-------------|-------------|-------|----------------|
| CONV-01 | Conversation context across messages | Phase 2 | Valkey session memory with 1hr TTL, last 10-20 messages |
| NLU-01 | Natural language understanding | Phase 2 | Claude Haiku 4.5 via n8n AI Agent node |
| CRM-01 | CRM queries (pipeline, leads, loans, partners) | Phase 1 (API), Phase 2 (query) | API endpoints in Phase 1, query capability in Phase 2 |
| TASK-01 | Multi-step tasks with approval | Phase 3 | Inline keyboard approval workflow, state machine |
| UX-01 | Contextual quick-action buttons | Phase 2 | InlineKeyboardMarkup on every response |
| ROUTE-01 | 80/20 rule-based vs AI routing | Phase 2 | Switch node rules first, Claude AI Agent fallback |
| EMAIL-01 | Outlook email scanning and drafting | Phase 5 | Microsoft Graph API, approval-gated sending |
| VOICE-01 | Voice-to-text input | Phase 6 | Deepgram Nova-3 transcription, feeds into text pipeline |
| LEARN-01 | Learn user preferences over time | Phase 6 | PostgreSQL ai_memory table, FACT/PREFERENCE/CORRECTION types |
| LOG-01 | Compliance logging | Phase 1 | bot_logs table, every message and tool call logged |

**Coverage: 10/10 v1 requirements mapped. No orphans.**

## Risk Mitigation

| Risk | Phase | Mitigation |
|------|-------|------------|
| Telegram single-webhook constraint | Phase 1 | ONE workflow with internal Switch routing -- never multiple Telegram Triggers |
| n8n memory pollution bug (#22112) | Phase 2 | Post-processing Code node strips large tool outputs before Valkey save |
| Claude hallucinating CRM data | Phase 2 | System prompt guardrails: "NEVER guess numbers/dates. If tool fails, say so." |
| Webhook timeout causing duplicates | Phase 1 | Acknowledge immediately with typing indicator, track processed update_ids |
| API cost spiral | Phase 2 | Rule engine handles 80% at $0; Haiku primary ($1/$5 per MTok); Sonnet only for complex |
| Stale approval execution | Phase 3 | 15-minute TTL on pending approvals; unique action_id in callback data |
| Bot token exposure | Phase 1 | Doppler only; .gitignore patterns; log sanitization |
| Valkey-n8n networking | Phase 1 | Verify same VPC or configure public access during provisioning |

## Progress

**Execution Order:**
Phases execute in numeric order: 1 -> 2 -> 3 -> 4 -> 5 -> 6
Note: Phase 4 depends on Phase 2 (not Phase 3), so Phases 3 and 4 could theoretically run in parallel.

| Phase | Plans Complete | Status | Completed |
|-------|---------------|--------|-----------|
| 1. Infrastructure & Foundation | 0/2 | Not started | - |
| 2. Core Query Assistant | 0/3 | Not started | - |
| 3. Write Operations & Approvals | 0/2 | Not started | - |
| 4. Proactive Alerts & Scheduling | 0/2 | Not started | - |
| 5. Outlook Integration | 0/2 | Not started | - |
| 6. Voice Input & Persistent Learning | 0/2 | Not started | - |
