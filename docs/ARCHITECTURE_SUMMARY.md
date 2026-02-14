# Telegram AI Assistant - Architecture Summary

**Date**: 2026-02-14
**Quick Reference**: Visual overview and key decisions

---

## 🎨 System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        USER (Harris)                             │
│                    Telegram: @Bigterrys_bot                      │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    TELEGRAM BOT API                              │
│                  (Message Gateway)                               │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                  n8n ORCHESTRATION LAYER                         │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              COMMAND ROUTER WORKFLOW                      │  │
│  │                                                            │  │
│  │  1. Load Session Memory (Redis)                          │  │
│  │  2. Check Rule Patterns (Fast Path)                      │  │
│  │  3. IF no match → Claude Intent Classification           │  │
│  │  4. Route to appropriate handler                         │  │
│  └──────────────┬──────────────────┬────────────────────────┘  │
│                 │                  │                             │
│                 ▼                  ▼                             │
│  ┌──────────────────┐  ┌──────────────────────────────┐        │
│  │  Rule-Based      │  │  AI-Powered Handler          │        │
│  │  Handler         │  │                               │        │
│  │  (80% queries)   │  │  [Claude Intent Node]        │        │
│  │                  │  │  - Analyze message            │        │
│  │  Direct API call │  │  - Determine tool(s)          │        │
│  │  Simple response │  │  - Generate response          │        │
│  └──────────────────┘  └──────────┬───────────────────┘        │
│                                    │                             │
│                                    ▼                             │
│                         ┌──────────────────────┐                │
│                         │   TOOL EXECUTOR      │                │
│                         │   - Parse function   │                │
│                         │   - Call API         │                │
│                         │   - Format result    │                │
│                         └──────────┬───────────┘                │
└────────────────────────────────────┼────────────────────────────┘
                                     │
                     ┌───────────────┼───────────────┐
                     │               │               │
                     ▼               ▼               ▼
        ┌──────────────────┐ ┌──────────────┐ ┌──────────────┐
        │   CRM APIs       │ │  Outlook API │ │  Calendar    │
        │                  │ │  (Microsoft  │ │  API         │
        │  - Loans         │ │   Graph)     │ │              │
        │  - Leads         │ │              │ │              │
        │  - Partners      │ │  - Email     │ │  - Check     │
        │  - Analytics     │ │  - Drafts    │ │  - Create    │
        └────────┬─────────┘ └──────────────┘ └──────────────┘
                 │
                 ▼
        ┌──────────────────────────────────────────────┐
        │           PostgreSQL Database                 │
        │                                               │
        │  Tables:                                      │
        │  - users, loans, leads, partners, activities │
        │  - ai_memory (NEW: persistent memory)        │
        │  - webhook_logs (integration tracking)       │
        └───────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    MEMORY SYSTEMS                                │
│                                                                   │
│  ┌──────────────────────┐      ┌─────────────────────────────┐ │
│  │  Session Memory      │      │  Persistent Memory          │ │
│  │  (Redis)             │      │  (PostgreSQL)               │ │
│  │                      │      │                             │ │
│  │  - Recent messages   │      │  - User preferences         │ │
│  │  - Current context   │      │  - Historical facts         │ │
│  │  - Pending approvals │      │  - Learned patterns         │ │
│  │  - TTL: 1 hour       │      │  - Semantic search (vector) │ │
│  └──────────────────────┘      └─────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔀 Message Flow Examples

### Example 1: Simple Query (Rule-Based - Fast)

```
User: "pipeline"
    ↓
Rule Match: "pipeline" → get_active_loans
    ↓
Direct API: GET /api/assistant/loans/active
    ↓
Template Response: "You have 8 active loans totaling $3.7M..."
    ↓
Telegram Reply (< 500ms)
```

### Example 2: Complex Query (AI-Powered)

```
User: "Are any VA loans at risk of missing closing dates?"
    ↓
No Rule Match → Claude Intent Classification
    ↓
Claude Function Call: get_active_loans(filter: "VA", risk_analysis: true)
    ↓
CRM API: Returns VA loans with dates
    ↓
Claude Analysis: Checks lock dates, closing dates, contingencies
    ↓
Claude Response: "Kevin's VA loan has a tight timeline..."
    ↓
Telegram Reply (2-3 seconds)
```

### Example 3: Multi-Step Task (Orchestrated)

```
User: "Check my calendar and email Joe if I'm free tomorrow at 10"
    ↓
Claude Plans Task:
  Step 1: check_calendar(date: "2026-02-15", time: "10:00")
  Step 2: IF available → draft_email(to: "joe@...")
  Step 3: request_approval()
    ↓
n8n Executes Steps:
  1. Outlook API → "You're free"
  2. Draft email → Saved as draft
  3. Send approval request → "Send this email? [Yes] [No] [Edit]"
    ↓
User: "yes"
    ↓
Send email via Microsoft Graph
    ↓
Telegram: "Done! Joe should get it any moment."
```

---

## 🔑 Key Architectural Decisions

### 1. Hybrid Intent Recognition (Rule + AI)

**Decision**: Use rule-based routing for 80% of common queries, Claude for 20% edge cases

**Why**:
- **Speed**: Rules execute in <100ms, no LLM latency
- **Cost**: No API tokens for routine queries
- **Reliability**: Predictable responses for common tasks
- **Flexibility**: Claude handles anything unexpected

**Trade-off**: Must maintain rule patterns as usage evolves

---

### 2. Two-Tier Memory (Session + Persistent)

**Decision**: Redis for short-term, PostgreSQL for long-term

**Why**:
- **Performance**: Redis sub-millisecond reads for recent context
- **Persistence**: PostgreSQL never loses important facts
- **Semantic Search**: pgvector enables "find similar" queries
- **Cost**: Redis only stores active sessions (auto-expire)

**Trade-off**: Slightly more complex to manage two data stores

---

### 3. Tool Calling over Prompting

**Decision**: Use Claude's native function calling for actions

**Why**:
- **Structured**: Validated inputs, typed outputs
- **Reliable**: Less prone to hallucination than pure prompting
- **Debuggable**: Clear logs of what was called
- **Extensible**: Easy to add new tools

**Trade-off**: Requires defining tool schemas upfront

---

### 4. n8n Orchestration over Custom Code

**Decision**: Use n8n workflows instead of Python/Node backend

**Why**:
- **Visual**: Easy to understand flow at a glance
- **No-Code Edits**: User can tweak workflows without coding
- **Built-in Integrations**: 1000+ node library
- **Already Deployed**: Existing infrastructure ($49/mo)

**Trade-off**: Less flexibility than custom code for complex logic

---

### 5. Approval Workflows for Critical Actions

**Decision**: Require user approval for emails, calendar events, status updates

**Why**:
- **Safety**: Prevent AI mistakes from going public
- **Trust**: User sees what AI will do before it happens
- **Learning**: User feedback improves AI over time

**Trade-off**: Adds interaction step (not fully autonomous)

---

## 📊 80/20 Rule Implementation

### 80% of Queries (Rule-Based)

**Fast Path**: Direct API calls, templated responses

| Query | Rule Pattern | Handler | Response Time |
|-------|-------------|---------|---------------|
| "pipeline" | `/^(pipeline\|loans)$/i` | `GET /api/assistant/loans/active` | <500ms |
| "leads" | `/^(leads\|new leads)$/i` | `GET /api/assistant/leads/recent` | <500ms |
| "calendar" | `/^(calendar\|schedule)$/i` | `check_calendar()` | <500ms |
| "email" | `/^(email\|inbox)$/i` | `scan_emails()` | <1s |
| "help" | `/^(help\|\?)$/i` | Show menu | <100ms |

### 20% of Queries (AI-Powered)

**Flexible Path**: Claude analyzes intent, calls tools, generates response

| Query Example | Claude Actions |
|---------------|---------------|
| "Which loans are at risk?" | 1. get_active_loans()<br>2. Analyze dates/statuses<br>3. Generate risk assessment |
| "How's Kevin doing?" | 1. Determine if loan/lead/partner<br>2. get_loan_details(name="Kevin")<br>3. Summarize status |
| "Email Sarah if appraisal is done" | 1. get_loan_details(name="Sarah")<br>2. Check appraisal status<br>3. IF done → draft_email()<br>4. Request approval |

---

## 🛠️ Tool Registry (Phase 1)

### Core Tools (10 tools)

| Category | Tool | Description | Endpoint |
|----------|------|-------------|----------|
| **Loans** | `get_active_loans` | Get pipeline overview | `GET /api/assistant/loans/active` |
|  | `get_loan_details` | Get specific loan | `GET /api/assistant/loans/:id` |
|  | `update_loan_status` | Update status | `PUT /api/assistant/loans/:id/status` |
| **Leads** | `get_new_leads` | Get recent leads | `GET /api/assistant/leads/recent` |
|  | `log_lead_contact` | Log follow-up | `POST /api/assistant/leads/:id/contact` |
| **Calendar** | `check_availability` | Check calendar | `POST /api/assistant/calendar/check` |
|  | `create_event` | Schedule meeting | `POST /api/assistant/calendar/create` |
| **Email** | `scan_emails` | Summarize emails | `POST /api/assistant/email/scan` |
|  | `draft_email` | Create draft | `POST /api/assistant/email/draft` |
| **Partners** | `get_top_partners` | Get partner list | `GET /api/assistant/partners/top` |

---

## 💾 Memory System Design

### Session Memory (Redis)

**Key Structure**: `telegram_session:{chat_id}`
**TTL**: 3600 seconds (1 hour)

```json
{
  "messages": [
    {"role": "user", "content": "What's my pipeline?", "timestamp": "2026-02-14T10:00:00Z"},
    {"role": "assistant", "content": "You have 8 active loans...", "timestamp": "2026-02-14T10:00:05Z"}
  ],
  "context": {
    "last_topic": "pipeline",
    "mentioned_entities": ["Kevin", "Sarah"],
    "pending_approvals": []
  }
}
```

### Persistent Memory (PostgreSQL)

**Table**: `ai_memory`

```sql
CREATE TABLE ai_memory (
  id UUID PRIMARY KEY,
  user_id VARCHAR(255),        -- Telegram chat_id
  memory_type VARCHAR(50),     -- "FACT", "PREFERENCE", "TASK", "CORRECTION"
  key VARCHAR(255),            -- e.g., "email_preference"
  value TEXT,                  -- e.g., "Prefers morning summaries"
  metadata JSONB,              -- Additional context
  embedding VECTOR(1536),      -- For semantic search
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);
```

**Example Memories**:

| Type | Key | Value |
|------|-----|-------|
| PREFERENCE | `email_style` | "Prefers morning email summaries with bullet points" |
| FACT | `kevin_folsomtrexler_loan` | "Kevin's $500k refi closes on 2026-02-17" |
| TASK | `reminded_lock_expiration` | "Reminded about Kevin's lock expiring on 2026-02-14" |
| CORRECTION | `sarah_loan_amount` | "User corrected: Sarah's loan is $525k, not $500k" |

---

## 🚀 Implementation Phases (Summary)

| Phase | Focus | Time | Priority |
|-------|-------|------|----------|
| **1** | Memory System (Redis + PostgreSQL) | 2-3 hrs | P1 |
| **2** | CRM API Endpoints (10 endpoints) | 3-4 hrs | P1 |
| **3** | Tool Registry & Execution | 2-3 hrs | P2 |
| **4** | Intent Classification | 2-3 hrs | P2 |
| **5** | Response Generation | 1-2 hrs | P3 |
| **6** | Multi-Step Orchestration | 2-3 hrs | P3 |

**Total Time**: 12-18 hours (1.5-2 weeks part-time)

---

## ✅ Success Metrics

### Technical Metrics
- ✅ **Response Time**: <3 seconds for 90% of queries
- ✅ **Accuracy**: 95%+ correct intent classification
- ✅ **Uptime**: 99%+ (n8n monitoring)
- ✅ **Error Rate**: <1% (logged to webhook_logs)

### User Experience Metrics
- ✅ **Naturalness**: 4+/5 rating on conversational feel
- ✅ **Task Completion**: 90%+ multi-step tasks succeed
- ✅ **Time Saved**: 2-3 hours/week vs manual CRM usage

### Business Metrics
- ✅ **Faster Response**: Instant answers vs opening web CRM
- ✅ **Better Follow-up**: Proactive reminders prevent missed deadlines
- ✅ **Mobile-First**: Can manage business from phone

---

## 💰 Cost Breakdown

| Component | Monthly Cost | Status |
|-----------|-------------|--------|
| n8n (Digital Ocean) | $49.00 | ✅ Existing |
| PostgreSQL (Managed) | $15.15 | ✅ Existing |
| Redis (Managed) | $10.00 | 🔨 New |
| Claude API | $20-40 | 🔨 New |
| Telegram API | $0.00 | Free |
| Microsoft Graph API | $0.00 | Included in Office 365 |
| **Total** | **$94-114/mo** | |

**ROI**: Saves 8-12 hours/month ($200-600 value) → **Positive ROI**

---

## 📚 Documentation References

- **Full Architecture**: `AI_ASSISTANT_ARCHITECTURE.md` (15,000 words)
- **Implementation Plan**: `IMPLEMENTATION_PLAN.md` (detailed steps)
- **Setup Progress**: `SETUP_PROGRESS.md` (current status)
- **Tool Registry**: `tools/registry.json` (to be created)

---

## 🎯 Next Immediate Steps

1. **Review this architecture** with user
2. **Get approval** on key decisions (hybrid intent, two-tier memory, tool calling)
3. **Create `ai_memory` table** (Prisma migration)
4. **Set up Redis** (Digital Ocean or managed service)
5. **Build first CRM API endpoints** (loans, leads)

**Time to First Working Demo**: ~4-5 hours

---

**Document Version**: 1.0
**Last Updated**: 2026-02-14
**Status**: Ready for user review
