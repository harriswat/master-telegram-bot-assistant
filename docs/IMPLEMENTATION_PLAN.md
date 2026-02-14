# Telegram AI Assistant - Implementation Plan

**Date**: 2026-02-14
**Status**: Ready to implement
**Based on**: AI_ASSISTANT_ARCHITECTURE.md

---

## 🎯 Goal

Transform the basic Telegram bot (70% complete) into a **conversational AI assistant** that:
- Understands natural language (not just commands)
- Maintains context across conversations
- Handles multi-step tasks automatically
- Feels natural and human-like

---

## 📋 Prerequisites (Already Complete)

✅ Telegram bot configured (`@Bigterrys_bot`)
✅ n8n workflows imported (3 of 4)
✅ Claude API connected
✅ Outlook OAuth working
✅ Google Sheets integration
✅ PostgreSQL database available
✅ Doppler secrets configured (21 secrets)

---

## 🏗️ Implementation Phases

### Phase 1: Memory System (Priority 1)
**Estimated Time**: 2-3 hours
**Goal**: Enable the assistant to remember context and learn preferences

#### 1.1 Set Up Redis for Session Memory
**Why**: Fast access to recent conversation history

**Tasks**:
- [ ] Install Redis on Digital Ocean (or use managed Redis)
- [ ] Add Redis credentials to Doppler
- [ ] Create n8n Redis credential
- [ ] Test connection

**n8n Workflow**:
```
Telegram Message Received
    ↓
Load Session Memory (Redis GET)
    ↓
Process Message with Context
    ↓
Save Updated Memory (Redis SET with 1hr TTL)
```

**Redis Key Structure**:
```javascript
{
  key: "telegram_session:8381242181",
  ttl: 3600, // 1 hour
  value: {
    messages: [
      {"role": "user", "content": "What's my pipeline?", "timestamp": "2026-02-14T10:00:00Z"},
      {"role": "assistant", "content": "You have $3.7M in active loans...", "timestamp": "2026-02-14T10:00:05Z"}
    ],
    context: {
      last_topic: "pipeline",
      mentioned_entities: ["Kevin Folsomtrexler", "Sarah Martinez"],
      pending_approvals: []
    }
  }
}
```

#### 1.2 Create Persistent Memory Table (PostgreSQL)
**Why**: Long-term memory for facts, preferences, historical context

**Tasks**:
- [ ] Create Prisma migration for `ai_memory` table
- [ ] Add pgvector extension for semantic search
- [ ] Create indexes for performance
- [ ] Test memory insert/retrieve

**Prisma Schema** (add to `schema.prisma`):
```prisma
model AiMemory {
  id          String   @id @default(cuid())
  userId      String   // Telegram chat_id
  memoryType  MemoryType
  key         String
  value       String   @db.Text
  metadata    Json?
  embedding   Unsupported("vector(1536)")? // pgvector for semantic search
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  expiresAt   DateTime?

  @@index([userId])
  @@index([memoryType])
  @@map("ai_memory")
}

enum MemoryType {
  FACT          // "User prefers morning email summaries"
  PREFERENCE    // "Use casual tone"
  TASK          // "Reminded about Kevin's lock expiration"
  CORRECTION    // "User corrected loan amount"
  CONVERSATION  // Compressed old conversation summary
}
```

**Migration Command**:
```bash
npx prisma migrate dev --name add_ai_memory
```

#### 1.3 Build Memory Management n8n Workflow
**Tasks**:
- [ ] Create "Memory Manager" sub-workflow
- [ ] Add nodes for: Load Session, Save Session, Store Fact, Retrieve Facts
- [ ] Test memory persistence

---

### Phase 2: Core CRM API Endpoints (Priority 1)
**Estimated Time**: 3-4 hours
**Goal**: Provide structured data access for AI assistant

#### 2.1 Loan Management APIs

**Endpoint 1**: `GET /api/assistant/loans/active`
```typescript
// Returns active loans with filters
export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const filter = searchParams.get('filter'); // "all", "closing_soon", "lock_expiring"

  // Query based on filter
  const loans = await prisma.loan.findMany({
    where: {
      status: { in: ['APPLICATION', 'PROCESSING', 'UNDERWRITING', 'CONDITIONAL', 'CLEAR_TO_CLOSE', 'DOCS_OUT'] },
      // Add filter logic
    },
    include: {
      borrower: true,
      loanOfficer: true,
      referralPartner: true
    },
    orderBy: { closingDate: 'asc' }
  });

  return Response.json({ loans, total: loans.length });
}
```

**Endpoint 2**: `GET /api/assistant/loans/:id`
```typescript
// Returns detailed loan information
```

**Endpoint 3**: `PUT /api/assistant/loans/:id/status`
```typescript
// Updates loan status with activity logging
```

#### 2.2 Lead Management APIs

**Endpoint 4**: `GET /api/assistant/leads/recent`
```typescript
// Returns recent leads (last 7 days by default)
```

**Endpoint 5**: `POST /api/assistant/leads/:id/contact`
```typescript
// Logs lead contact attempt
```

#### 2.3 Calendar & Email APIs

**Endpoint 6**: `POST /api/assistant/calendar/check`
```typescript
// Check Outlook calendar availability (proxied to Microsoft Graph)
```

**Endpoint 7**: `POST /api/assistant/email/scan`
```typescript
// Scan recent emails and return summary
```

#### 2.4 Partner APIs

**Endpoint 8**: `GET /api/assistant/partners/top`
```typescript
// Returns top referral partners by tier
```

**Files to Create**:
```
/src/app/api/assistant/
  ├── loans/
  │   ├── active/route.ts
  │   ├── [id]/route.ts
  │   └── [id]/status/route.ts
  ├── leads/
  │   ├── recent/route.ts
  │   └── [id]/contact/route.ts
  ├── calendar/
  │   └── check/route.ts
  ├── email/
  │   └── scan/route.ts
  └── partners/
      └── top/route.ts
```

**Testing**:
- [ ] Test all endpoints with Postman/curl
- [ ] Verify role-based access control
- [ ] Ensure proper error handling

---

### Phase 3: Tool Registry & Function Calling (Priority 2)
**Estimated Time**: 2-3 hours
**Goal**: Define structured tools that Claude can call

#### 3.1 Create Tool Definitions JSON

**File**: `integrations/telegram-ai-assistant/tools/registry.json`

```json
{
  "tools": [
    {
      "name": "get_active_loans",
      "description": "Retrieves active loans from the CRM pipeline. Use this when the user asks about their pipeline, active loans, or loans closing soon.",
      "input_schema": {
        "type": "object",
        "properties": {
          "filter": {
            "type": "string",
            "enum": ["all", "closing_this_week", "lock_expiring_soon"],
            "description": "Filter criteria for loans"
          },
          "limit": {
            "type": "integer",
            "default": 10,
            "description": "Maximum number of loans to return"
          }
        },
        "required": []
      },
      "endpoint": "/api/assistant/loans/active",
      "method": "GET"
    },
    {
      "name": "get_loan_details",
      "description": "Get detailed information about a specific loan. Use this when the user asks about a specific borrower or loan.",
      "input_schema": {
        "type": "object",
        "properties": {
          "borrower_name": {
            "type": "string",
            "description": "Name of the borrower (first or last name)"
          }
        },
        "required": ["borrower_name"]
      },
      "endpoint": "/api/assistant/loans/search",
      "method": "GET"
    },
    {
      "name": "check_calendar_availability",
      "description": "Check if the user is available on their calendar at a specific date/time. Use this when the user asks about their schedule or availability.",
      "input_schema": {
        "type": "object",
        "properties": {
          "date": {
            "type": "string",
            "description": "Date in YYYY-MM-DD format"
          },
          "time": {
            "type": "string",
            "description": "Time in HH:MM format (optional, checks whole day if not provided)"
          }
        },
        "required": ["date"]
      },
      "endpoint": "/api/assistant/calendar/check",
      "method": "POST"
    },
    {
      "name": "get_new_leads",
      "description": "Get recent leads from the CRM. Use this when the user asks about new leads or lead activity.",
      "input_schema": {
        "type": "object",
        "properties": {
          "days": {
            "type": "integer",
            "default": 7,
            "description": "Number of days to look back (default 7)"
          },
          "status": {
            "type": "string",
            "enum": ["NEW", "CONTACTED", "WORKING", "QUALIFIED"],
            "description": "Filter by lead status (optional)"
          }
        },
        "required": []
      },
      "endpoint": "/api/assistant/leads/recent",
      "method": "GET"
    },
    {
      "name": "scan_emails",
      "description": "Scan recent emails from Outlook and provide a summary. Use this when the user asks to check their email or wants an email summary.",
      "input_schema": {
        "type": "object",
        "properties": {
          "hours": {
            "type": "integer",
            "default": 24,
            "description": "Number of hours to look back"
          },
          "filter": {
            "type": "string",
            "enum": ["all", "unread", "important"],
            "description": "Email filter"
          }
        },
        "required": []
      },
      "endpoint": "/api/assistant/email/scan",
      "method": "POST"
    }
  ]
}
```

#### 3.2 Create Tool Execution n8n Workflow

**Workflow**: "Tool Executor"

**Flow**:
```
Function Call from Claude
    ↓
Parse Tool Name & Parameters
    ↓
[Switch Node: Route by Tool Name]
    ├─→ get_active_loans → HTTP Request to /api/assistant/loans/active
    ├─→ check_calendar → HTTP Request to /api/assistant/calendar/check
    ├─→ scan_emails → HTTP Request to /api/assistant/email/scan
    └─→ [Other tools...]
    ↓
Format Response for Claude
    ↓
Return Tool Result
```

**Error Handling**:
- Retry failed API calls (max 3 attempts)
- Log errors to webhook_logs table
- Return user-friendly error messages

---

### Phase 4: Intent Classification & Routing (Priority 2)
**Estimated Time**: 2-3 hours
**Goal**: Intelligently route user messages to appropriate handlers

#### 4.1 Update Command Router Workflow

**Add Intent Classification Node**:

**Flow**:
```
Telegram Message
    ↓
Load Session Memory (Redis)
    ↓
[IF: Matches Rule Pattern]
    YES → Direct API Call (fast path)
    NO → Claude Intent Classification
    ↓
[Claude Node: Classify Intent]
    → Returns: intent + tool_calls + confidence
    ↓
[IF: Tool Call Required]
    YES → Execute Tool → Format Response → Send to Telegram
    NO → Generate Conversational Response → Send to Telegram
    ↓
Save Session Memory (Redis)
```

**Example Rule Patterns** (fast path):
```javascript
const rules = [
  { pattern: /^(pipeline|loans)$/i, action: 'get_active_loans' },
  { pattern: /^(leads|new leads)$/i, action: 'get_new_leads' },
  { pattern: /^(calendar|schedule)$/i, action: 'check_calendar' },
  { pattern: /^(email|inbox)$/i, action: 'scan_emails' },
  { pattern: /^(help|\?)$/i, action: 'show_help' }
];
```

#### 4.2 Create Claude Intent Classification Prompt

**System Prompt**:
```
You are an AI assistant for Harris Watkins, a mortgage loan officer at Haven Home Loans LLC.

Your role is to:
1. Understand what Harris is asking for
2. Determine which tools (if any) need to be called
3. Provide helpful, conversational responses

Available tools:
{{TOOL_REGISTRY}}

Recent conversation:
{{SESSION_MEMORY}}

Relevant facts from history:
{{PERSISTENT_MEMORY}}

Harris's message: "{{USER_MESSAGE}}"

Analyze the intent and respond with:
1. If a tool call is needed, call the appropriate function
2. If no tool is needed, provide a conversational response
3. If the request is unclear, ask a clarifying question
```

---

### Phase 5: Response Generation (Priority 3)
**Estimated Time**: 1-2 hours
**Goal**: Generate natural, conversational responses

#### 5.1 Create Response Formatter Workflow

**Flow**:
```
Tool Result(s)
    ↓
[Claude Node: Format Response]
    → System: "Format this data into a natural, conversational message"
    → Context: {{TOOL_RESULTS}}
    → Style: "Be concise, friendly, and proactive"
    ↓
Add Proactive Suggestions
    ↓
Send to Telegram
```

**Example Response Templates**:

**Pipeline Query**:
```
Input: {"loans": 8, "totalValue": 3750000, "closingSoon": 2}
Output: "You have 8 active loans totaling $3.7M. Two are closing this week - Kevin's $500k refi on Thursday and Sarah's $525k purchase on Friday. Want me to check on their status?"
```

**Email Scan**:
```
Input: {"unread": 5, "important": 2, "summary": "..."}
Output: "You have 5 unread emails. Two look important:
1. Kevin Folsomtrexler asking about closing date
2. Sarah Martinez requesting rate lock extension

Want me to draft replies?"
```

---

### Phase 6: Multi-Step Task Orchestration (Priority 3)
**Estimated Time**: 2-3 hours
**Goal**: Handle complex, multi-step tasks automatically

#### 6.1 Add Task Planner Node

**When to trigger**: Claude returns multiple tool calls or conditional logic

**Example Task Plan**:
```json
{
  "task": "Check calendar and email partner if available",
  "steps": [
    {
      "step": 1,
      "tool": "check_calendar",
      "params": {"date": "2026-02-15", "time": "14:00"}
    },
    {
      "step": 2,
      "condition": "if calendar.available === true",
      "tool": "draft_email",
      "params": {
        "to": "emily@example.com",
        "subject": "Meeting Tomorrow",
        "content": "Hi Emily, I'm free tomorrow at 2 PM..."
      }
    },
    {
      "step": 3,
      "action": "request_approval",
      "message": "I've drafted an email to Emily. Send it? (yes/no/edit)"
    }
  ]
}
```

#### 6.2 Add Approval Workflow

**Trigger**: Multi-step task requires user approval

**Flow**:
```
Task Requires Approval
    ↓
Send Approval Request to Telegram
    (with inline buttons: Approve / Edit / Cancel)
    ↓
User Response
    ├─→ Approve → Execute Action → Notify User
    ├─→ Edit → Request Changes → Re-draft → Request Approval
    └─→ Cancel → Abort Task → Notify User
```

---

## 🧪 Testing Plan

### Unit Tests (n8n)

**Test 1: Memory Load/Save**
- [ ] Send message, verify Redis storage
- [ ] Load session, verify context preserved

**Test 2: Tool Execution**
- [ ] Call each tool with test parameters
- [ ] Verify correct API endpoint hit
- [ ] Verify response format

**Test 3: Intent Classification**
- [ ] Send ambiguous message, verify Claude classifies correctly
- [ ] Send clear command, verify rule match (fast path)

### Integration Tests (End-to-End)

**Test 1: Simple Query**
```
User: "What's my pipeline?"
Expected: Summary of active loans (<3 seconds)
```

**Test 2: Complex Query**
```
User: "Which VA loans are closing this month?"
Expected: Filtered list of VA loans with closing dates
```

**Test 3: Multi-Step Task**
```
User: "Check my calendar tomorrow and email Joe if I'm free at 10am"
Expected:
  1. Calendar checked
  2. Email drafted (if available)
  3. Approval requested
```

**Test 4: Context Carryover**
```
User: "What's my pipeline?"
Assistant: "You have 8 active loans..."
User: "How many are VA?"
Expected: Claude remembers we're talking about pipeline
```

### User Acceptance Testing

- [ ] User sends 10 realistic queries
- [ ] User rates naturalness of responses (1-5)
- [ ] User confirms all tool calls worked correctly

---

## 📊 Success Criteria

### Functional Requirements
- ✅ Memory system working (session + persistent)
- ✅ All core tools functional (10+ tools)
- ✅ Intent classification accurate (>95%)
- ✅ Multi-step tasks complete successfully
- ✅ Context carryover working

### Non-Functional Requirements
- ✅ Response time <3 seconds for 90% of queries
- ✅ Uptime >99% (n8n monitoring)
- ✅ Natural conversation feel (user feedback 4+/5)

---

## 🚀 Deployment Checklist

### Pre-Deployment
- [ ] All CRM API endpoints tested
- [ ] Memory system tested
- [ ] Tool registry validated
- [ ] n8n workflows activated
- [ ] Error handling tested

### Deployment
- [ ] Deploy CRM API endpoints (already on Digital Ocean)
- [ ] Configure Redis credentials in n8n
- [ ] Update Telegram webhook URL (if needed)
- [ ] Activate all n8n workflows
- [ ] Test with user's Telegram account

### Post-Deployment
- [ ] Monitor error logs (webhook_logs table)
- [ ] Track response times (n8n metrics)
- [ ] Collect user feedback
- [ ] Iterate on prompts and tools

---

## 💰 Cost Estimate

### Infrastructure (Monthly)
- **Digital Ocean n8n**: $49/mo (existing)
- **PostgreSQL Database**: $15.15/mo (existing)
- **Redis (if managed)**: ~$10/mo (optional, can use DO Redis)
- **Total Infrastructure**: ~$74/mo

### API Costs (Estimated)
- **Claude API**:
  - ~1000 requests/day (30K/month)
  - Avg 2K tokens/request (mix of Haiku + Sonnet)
  - Cost: ~$20-40/month
- **Microsoft Graph API**: Free (Office 365 subscription)
- **Telegram API**: Free

**Total Estimated Cost**: ~$94-114/month

**ROI**: Saves 2-3 hours/week = 8-12 hours/month
- Value: $100-300/month (at $25-50/hr)
- **Positive ROI** even at low end

---

## 📅 Timeline

### Week 1 (Now)
- **Day 1-2**: Phase 1 (Memory System)
- **Day 3-4**: Phase 2 (CRM APIs)
- **Day 5**: Phase 3 (Tool Registry)

### Week 2
- **Day 1-2**: Phase 4 (Intent Classification)
- **Day 3**: Phase 5 (Response Generation)
- **Day 4-5**: Testing & Refinement

### Week 3
- **Day 1-2**: Phase 6 (Multi-Step Tasks)
- **Day 3-5**: User Testing & Iteration

**Launch Target**: End of Week 3 (Feb 28, 2026)

---

## 🎯 Immediate Next Steps

1. **Create `ai_memory` Prisma migration** (30 minutes)
2. **Set up Redis** (1 hour)
3. **Build first 3 CRM API endpoints** (2 hours)
4. **Test basic tool calling** (1 hour)

**Total Time to First Working Prototype**: ~4.5 hours

---

## 📚 Resources

- Architecture Document: `AI_ASSISTANT_ARCHITECTURE.md`
- Tool Registry: `tools/registry.json` (to be created)
- n8n Workflows: `workflows/json/` (existing)
- CRM API Specs: `docs/API_REFERENCE.md` (to be created)

---

**Document Version**: 1.0
**Last Updated**: 2026-02-14
**Status**: Ready for implementation
