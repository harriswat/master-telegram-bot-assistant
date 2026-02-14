# Telegram AI Assistant Architecture - Comprehensive Design

**Date**: 2026-02-14
**Project**: Haven Home Loans CRM - Telegram AI Executive Assistant
**Goal**: Build a conversational AI assistant that feels natural and human-like

---

## 🎯 Executive Summary

Based on 2026 industry best practices and analysis of real-world AI assistant architectures, this document outlines a **hybrid architecture** that combines:

1. **LLM-powered natural language understanding** (Claude API) for flexible, context-aware interpretation
2. **Structured tool/function calling** for reliable, deterministic actions
3. **Persistent memory** for long-term context and learning
4. **n8n orchestration layer** for workflow automation and integration
5. **Telegram as the conversational interface**

This architecture prioritizes **maintainability, extensibility, and natural conversation flow** over rigid command structures.

---

## 📚 Research Foundations

This architecture is informed by:

- [AI agent architecture patterns in 2026](https://www.lindy.ai/blog/ai-agent-architecture) (Lindy)
- [LLM chatbot architecture best practices](https://rasa.com/blog/llm-chatbot-architecture) (Rasa)
- [Voice AI stack for building agents](https://www.assemblyai.com/blog/the-voice-ai-stack-for-building-agents) (AssemblyAI)
- [AI agent memory systems](https://redis.io/blog/ai-agent-memory-stateful-systems/) (Redis)
- [Context window management strategies](https://www.getmaxim.ai/articles/context-window-management-strategies-for-long-context-ai-agents-and-chatbots) (Maxim)
- [n8n + Claude integration patterns](https://n8n.io/integrations/claude/)
- [Telegram bot AI architecture](https://danoncoding.com/building-an-ai-telegram-agent-with-python-and-claude-2f18a0d1a6dc)

---

## 🏗️ System Architecture

### High-Level Flow

```
User Message (Telegram)
    ↓
Telegram Bot API
    ↓
n8n Command Router Workflow
    ↓
Intent Classification (Claude API)
    ├─→ Structured Tool/Function Call
    │   └─→ Execute Action (CRM API, Outlook, Calendar, etc.)
    └─→ Conversational Response
        └─→ Claude generates natural language reply
    ↓
Response sent to Telegram
    ↓
Memory Update (session + persistent)
```

### Component Breakdown

#### 1. **Conversational Interface Layer** (Telegram)
- **What**: Telegram Bot API
- **Why**: Familiar, mobile-first, supports rich media (voice, photos, documents)
- **Status**: ✅ Already configured (`@Bigterrys_bot`)

#### 2. **Orchestration Layer** (n8n)
- **What**: Central workflow engine that routes messages, calls APIs, manages state
- **Why**: Visual workflow builder, easy to debug, no-code extensibility
- **Status**: ✅ Deployed on Digital Ocean ($49/mo)

#### 3. **Intelligence Layer** (Claude API)
- **What**: LLM for intent recognition, natural language understanding, response generation
- **Why**: State-of-the-art reasoning, tool calling, context understanding
- **Status**: ✅ API key configured

#### 4. **Memory Layer** (PostgreSQL + Redis)
- **What**:
  - **Session Memory**: Short-term context for active conversations (Redis)
  - **Persistent Memory**: Long-term facts, preferences, history (PostgreSQL)
- **Why**: Maintains context across conversations, learns user preferences
- **Status**: ⚠️ PostgreSQL available, Redis not yet configured

#### 5. **Tool/Action Layer** (CRM APIs, Microsoft Graph, Google Sheets)
- **What**: Structured functions the AI can call (get loans, check calendar, send email)
- **Why**: Reliable, validated actions with error handling
- **Status**: ⚠️ CRM API endpoints need to be built (Phase 12)

---

## 🧠 Intent Recognition Strategy

### Hybrid Approach: AI + Rules (Recommended)

**Why Hybrid?**
- LLMs alone can be unpredictable and slow
- Rules alone are brittle and don't scale
- Hybrid gets best of both worlds

### Implementation:

1. **Fast Rule-Based Pre-Filter** (n8n)
   - Match explicit commands: `/help`, `/status`, `/pipeline`
   - Match common patterns: "What's my pipeline?" → `get_pipeline()`
   - Bypass LLM for 80% of routine queries (fast + cheap)

2. **AI-Powered Intent Classification** (Claude)
   - Handle ambiguous queries: "How's that loan for Kevin looking?"
   - Extract entities: names, dates, amounts
   - Determine multi-step tasks: "Check my calendar and email Joe if I'm free tomorrow"

3. **Tool Calling** (Claude's native function calling)
   - Claude determines which tool(s) to call
   - n8n validates and executes the tool
   - Results fed back to Claude for natural response

### Example Flow:

**User**: "Do I have any loans closing this week?"

1. **Rule Check**: No exact match → pass to Claude
2. **Claude Intent**: `get_active_loans(filter: "closing_this_week")`
3. **n8n Execution**: Calls CRM API `/api/loans/active?closing_soon=true`
4. **Claude Response**: "Yes, you have 2 loans closing this week: Kevin Folsomtrexler ($500k) on Thursday and Sarah Martinez ($525k) on Friday."

---

## 🛠️ Tool/Function Architecture

### Core Tools (Phase 1)

Based on your CRM's capabilities and most common use cases:

#### **Loan Management**
- `get_active_loans(filter?)` - Get pipeline overview
- `get_loan_details(borrower_name)` - Get specific loan info
- `update_loan_status(loan_id, status)` - Update status
- `get_closing_schedule(days?)` - Get upcoming closings

#### **Lead Management**
- `get_new_leads(days?)` - Get recent leads
- `assign_lead(lead_id, loan_officer)` - Assign to LO
- `get_lead_details(name)` - Get lead info
- `log_lead_contact(lead_id, method, result)` - Log follow-up

#### **Calendar & Availability**
- `check_availability(date, time?)` - Check Outlook calendar
- `create_calendar_event(title, date, time, attendees)` - Schedule meeting
- `get_today_schedule()` - Get today's appointments

#### **Email & Communication**
- `scan_recent_emails(hours?)` - Summarize emails with AI
- `draft_email(to, subject, content)` - Create draft for approval
- `send_approved_email(draft_id)` - Send approved draft

#### **Partner Management**
- `get_top_partners(tier?)` - Get partner list
- `log_partner_contact(partner_id, type)` - Log communication
- `get_partner_performance(partner_name)` - Get metrics

#### **Analytics & Reporting**
- `get_pipeline_value()` - Total pipeline value
- `get_closed_volume(period?)` - Closed loan volume
- `get_commission_summary(period?)` - Commission earned (OWNER only)

### Tool Definition Format (Claude Function Calling)

```json
{
  "name": "get_active_loans",
  "description": "Retrieves active loans from the CRM pipeline with optional filters",
  "input_schema": {
    "type": "object",
    "properties": {
      "filter": {
        "type": "string",
        "enum": ["all", "closing_this_week", "lock_expiring_soon", "by_status"],
        "description": "Filter criteria for loans"
      },
      "status": {
        "type": "string",
        "description": "If filter is 'by_status', specify status (APPLICATION, PROCESSING, etc.)"
      },
      "limit": {
        "type": "integer",
        "default": 10,
        "description": "Maximum number of loans to return"
      }
    },
    "required": []
  }
}
```

---

## 💾 Memory Architecture

### Two-Tier Memory System

#### **Tier 1: Session Memory (Redis)** - Short-term, fast access
- Conversation history (last 10-20 messages)
- Current context (what we're talking about)
- Pending approvals (email drafts awaiting approval)
- User preferences for this session

**Implementation**:
```javascript
// n8n Redis node
{
  "key": "telegram_session:{chat_id}",
  "ttl": 3600, // 1 hour
  "value": {
    "messages": [
      {"role": "user", "content": "What's my pipeline?"},
      {"role": "assistant", "content": "You have $3.7M in active loans..."}
    ],
    "context": {
      "last_query": "pipeline",
      "mentioned_loans": ["loan_123", "loan_456"]
    },
    "pending_approvals": ["draft_789"]
  }
}
```

#### **Tier 2: Persistent Memory (PostgreSQL)** - Long-term, rich data
- User profile (preferences, communication style)
- Historical facts (birthdays, anniversaries mentioned)
- Task history (what's been asked before)
- Learning data (successful responses, corrections)

**New Table: `ai_memory`**
```sql
CREATE TABLE ai_memory (
  id UUID PRIMARY KEY,
  user_id VARCHAR(255) NOT NULL, -- Telegram chat_id
  memory_type VARCHAR(50) NOT NULL, -- "fact", "preference", "task", "correction"
  key VARCHAR(255) NOT NULL,
  value TEXT NOT NULL,
  metadata JSONB, -- Additional context
  embedding VECTOR(1536), -- For semantic search (pgvector extension)
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  expires_at TIMESTAMP NULL -- Optional expiry
);

CREATE INDEX idx_memory_user ON ai_memory(user_id);
CREATE INDEX idx_memory_type ON ai_memory(memory_type);
CREATE INDEX idx_memory_embedding ON ai_memory USING ivfflat (embedding vector_cosine_ops);
```

#### **Memory Retrieval Strategy**

1. **Session Recall** (Redis): Load last conversation (instant)
2. **Semantic Search** (PostgreSQL + pgvector): Find relevant facts from history
3. **Recency Boost**: Weight recent memories higher
4. **Summarization**: Compress old sessions into facts

**Example Memory Injection**:
```
System: You are Harris's executive assistant for his mortgage business.

Recent conversation:
- Harris asked about his pipeline (5 minutes ago)
- You told him he has $3.7M in active loans

Relevant facts from history:
- Harris prefers email summaries in the morning (learned 2024-01-15)
- He's working with Kevin Folsomtrexler on a $500k refi (mentioned yesterday)
- He has a weekly call with top partners on Fridays (recurring)
```

---

## 🎯 Making Conversations Feel Natural

### Key Principles

#### 1. **Avoid Rigid Commands**
❌ **Bad**: "Use `/email scan` to check your emails"
✅ **Good**: "I can check your emails for you. Want me to take a look?"

#### 2. **Use Conversational Language**
❌ **Bad**: "Loan status updated to PROCESSING"
✅ **Good**: "Got it! I've marked Kevin's loan as in processing."

#### 3. **Ask Clarifying Questions**
❌ **Bad**: "Error: Multiple loans found for 'Sarah'"
✅ **Good**: "I found two Sarahs - Martinez (purchase) and Thompson (refi). Which one?"

#### 4. **Proactive Suggestions**
✅ **Example**: "By the way, Kevin's rate lock expires in 3 days. Want me to remind him?"

#### 5. **Graceful Error Recovery**
❌ **Bad**: "Function call failed: TypeError"
✅ **Good**: "Hmm, I'm having trouble pulling up that loan. Let me try again... [retries] Okay, got it!"

#### 6. **Context Carryover**
**User**: "What's my pipeline?"
**Assistant**: "You have $3.7M in active loans across 8 deals."
**User**: "How many are closing this week?"
**Assistant**: "Two of those are closing this week..." ← Remembers "pipeline" context

---

## 🔄 Multi-Step Task Handling

### Orchestration Pattern: Plan → Execute → Verify

#### Example: "Check my calendar and email Joe if I'm free tomorrow"

**Step 1: Claude Plans the Task**
```json
{
  "task": "conditional_email_based_on_calendar",
  "steps": [
    {
      "step": 1,
      "tool": "check_availability",
      "params": {"date": "2026-02-15", "time": "10:00"}
    },
    {
      "step": 2,
      "condition": "if_available",
      "tool": "draft_email",
      "params": {
        "to": "joe@example.com",
        "subject": "Available tomorrow",
        "content": "Hi Joe, I'm free tomorrow at 10 AM. Want to meet?"
      }
    },
    {
      "step": 3,
      "tool": "send_approval_request",
      "params": {"draft_id": "{{step2.draft_id}}"}
    }
  ]
}
```

**Step 2: n8n Executes Each Step**
- Calls Outlook API to check calendar
- If available, generates email draft
- Sends approval request to Telegram

**Step 3: User Approval**
**Assistant**: "You're free tomorrow at 10 AM. I drafted an email to Joe suggesting a meeting. Send it? (yes/no/edit)"
**User**: "yes"
**Assistant**: "Sent! Joe should get it in a moment."

---

## 🧩 Extensibility Strategy

### Easy to Add New Capabilities

#### 1. **Define New Tool** (JSON)
```json
{
  "name": "get_referral_partner_stats",
  "description": "Get performance stats for a referral partner",
  "input_schema": {
    "type": "object",
    "properties": {
      "partner_name": {"type": "string"},
      "period": {"type": "string", "enum": ["month", "quarter", "year"]}
    },
    "required": ["partner_name"]
  }
}
```

#### 2. **Add n8n Workflow Node**
- Create HTTP Request node to CRM API
- Add error handling
- Format response for Claude

#### 3. **Update Tool Registry**
- Add tool definition to Claude's system prompt
- n8n automatically routes function calls

#### 4. **Test & Deploy**
- No code changes to main workflow
- New capability available instantly

---

## 📊 The 80/20 Rule Implementation

### 80% Common Tasks (Fast, Structured)

**Use rule-based routing:**
- "What's my pipeline?" → Direct API call, templated response
- "Check my calendar" → Outlook API, formatted list
- "New leads" → CRM API, simple list

**Benefits**:
- Fast (no LLM overhead)
- Cheap (no API tokens)
- Predictable (consistent format)

### 20% Edge Cases (Flexible, AI-Powered)

**Use Claude for:**
- Complex queries: "Which loans are at risk of falling out?"
- Multi-step tasks: "Find Sarah's loan, check if appraisal is done, and email her an update"
- Ambiguous input: "How's Kevin doing?" → Determine if it's a loan, lead, or partner

**Benefits**:
- Handles anything user throws at it
- Natural language understanding
- Can reason and make decisions

---

## 🚀 Recommended Implementation Phases

### Phase 1: Foundation (Week 1)
✅ **Complete** (mostly done):
- Telegram bot configured
- n8n workflows imported
- Credentials set up

### Phase 2: Memory System (Week 1-2)
⏳ **In Progress**:
- Set up Redis for session memory
- Create `ai_memory` table in PostgreSQL
- Implement memory load/save in n8n

### Phase 3: Core CRM API Endpoints (Week 2)
🔨 **Build**:
- `/api/loans/active` - Get pipeline
- `/api/leads/recent` - Get new leads
- `/api/partners/top` - Get partners
- All endpoints return JSON for n8n consumption

### Phase 4: Tool Integration (Week 2-3)
🔨 **Build**:
- Define 10-15 core tools (see "Core Tools" section above)
- Create n8n sub-workflows for each tool
- Add error handling and validation

### Phase 5: Conversational Layer (Week 3)
🔨 **Build**:
- Claude system prompt with tool definitions
- Intent classification workflow
- Response generation workflow
- Context injection from memory

### Phase 6: Multi-Step Tasks (Week 4)
🔨 **Build**:
- Task planning system
- Step-by-step execution
- Approval workflows
- Error recovery

### Phase 7: Learning & Refinement (Ongoing)
🔨 **Build**:
- User feedback collection
- Response quality tracking
- Memory optimization
- Tool usage analytics

---

## 🎨 Example Interactions

### Example 1: Simple Query (Rule-Based)

**User**: "pipeline"

**System**:
1. Rule matches "pipeline" keyword
2. Direct API call to `/api/loans/active`
3. Template response: "You have 8 active loans totaling $3.7M. 2 closing this week."

**Response Time**: <500ms

---

### Example 2: Complex Query (AI-Powered)

**User**: "Are any of my VA loans at risk of not closing on time?"

**System**:
1. No rule match → Claude analyzes intent
2. Claude function call:
   ```json
   {
     "tool": "get_active_loans",
     "params": {"filter": "by_product", "product": "VA"}
   }
   ```
3. Claude analyzes results:
   - Checks lock expiration dates
   - Checks closing dates
   - Identifies risks
4. Claude generates natural response:
   "You have 2 VA loans. Kevin's lock expires in 3 days and he closes in 5 days - pretty tight. Sarah's loan looks good though."

**Response Time**: 2-3 seconds

---

### Example 3: Multi-Step Task (AI Orchestration)

**User**: "Check my calendar and if I'm free Thursday at 2pm, schedule a call with Emily Rodriguez"

**System**:
1. Claude plans task:
   - Step 1: Check calendar (Thursday 2pm)
   - Step 2 (conditional): Create calendar event if free
   - Step 3: Notify user
2. n8n executes:
   - Outlook API → User is free
   - Create event: "Call with Emily Rodriguez"
3. Claude responds:
   "You're free Thursday at 2 PM! I've added a calendar event for your call with Emily. Want me to send her an email reminder?"

**Response Time**: 3-5 seconds

---

## ⚠️ Challenges & Solutions

### Challenge 1: LLM Latency (2-4 seconds)
**Solution**:
- Use rule-based routing for 80% of queries
- Implement streaming responses (show "typing..." indicator)
- Cache common responses

### Challenge 2: Cost (Claude API tokens)
**Solution**:
- Use Claude Haiku for intent classification (cheaper, faster)
- Use Claude Sonnet for response generation (better quality)
- Cache tool definitions (don't resend every time)

### Challenge 3: Context Window Limits
**Solution**:
- Summarize old conversations into facts
- Use semantic search to retrieve only relevant context
- Implement token budgets per section (max 1000 tokens for conversation history)

### Challenge 4: Hallucinations (AI making up data)
**Solution**:
- Always ground responses in tool outputs
- Never let Claude "guess" numbers or dates
- Validate all structured data before using

### Challenge 5: Error Recovery
**Solution**:
- Retry failed API calls automatically
- If tool fails, ask user to clarify
- Log all errors to webhook_logs table

---

## 🔐 Security Considerations

### Authentication
- **Telegram**: Whitelist specific chat IDs (only Harris can use bot)
- **n8n**: Basic auth + API key
- **CRM APIs**: JWT tokens with role-based access

### Data Privacy
- **No PII in logs**: Redact SSNs, full addresses from logs
- **Encrypted storage**: Redis and PostgreSQL encrypted at rest
- **Audit trail**: Log all actions to `audit_logs` table

### Rate Limiting
- **Telegram**: 30 messages/second (built-in)
- **Claude API**: 50 requests/minute (set in n8n)
- **CRM APIs**: 100 requests/minute per user

---

## 📈 Success Metrics

### User Experience
- **Response time**: <3 seconds for 90% of queries
- **Accuracy**: 95%+ correct intent classification
- **Completion rate**: 90%+ of multi-step tasks complete successfully

### Technical Performance
- **Uptime**: 99.9% (n8n monitoring)
- **Error rate**: <1% (webhook_logs tracking)
- **Memory efficiency**: <100ms to load context

### Business Impact
- **Time saved**: 2-3 hours/week on routine queries
- **Faster response**: Instant answers vs opening CRM
- **Better follow-up**: Proactive reminders prevent missed deadlines

---

## 🎓 Real-World Architecture References

### Similar Systems

1. **Lindy.ai** - AI executive assistant
   - Uses hybrid rule + AI approach
   - Memory-enhanced conversations
   - Tool calling for integrations

2. **Intercom's Fin** - Customer support AI
   - LLM-powered intent recognition
   - Structured actions for common queries
   - Escalation to humans when needed

3. **Clay's AI SDR** - Sales assistant
   - Research + outreach automation
   - Multi-step workflows
   - Approval workflows for emails

4. **Siri/Alexa Architecture**
   - Fast wake word detection (rules)
   - Intent classification (ML)
   - Skill routing (structured actions)

---

## 🔮 Future Enhancements (Post-MVP)

### Voice Support
- Integrate Deepgram for voice message transcription
- Voice notes → text → Claude processing

### Scheduled Tasks
- Morning briefing: "Good morning! Here's your day..."
- Lock expiration alerts: "Kevin's lock expires tomorrow!"
- Birthday reminders: "Sarah's birthday is next week"

### Multi-User Support
- Role-based access (OWNER, ADMIN, LOAN_OFFICER)
- Each user has own chat_id
- Shared memory for team context

### Advanced Analytics
- Conversation insights dashboard
- Tool usage heatmap
- Response quality tracking

### Integration Expansion
- Arive LOS integration (sync loan data)
- Zillow lead scraping
- SMS notifications via Twilio

---

## 📚 Sources & References

### Architecture & Best Practices
- [A Complete Guide to AI Agent Architecture in 2026](https://www.lindy.ai/blog/ai-agent-architecture) - Lindy
- [LLM Chatbot Architecture](https://rasa.com/blog/llm-chatbot-architecture) - Rasa
- [The Voice AI Stack for Building Agents](https://www.assemblyai.com/blog/the-voice-ai-stack-for-building-agents) - AssemblyAI
- [AI Agent Orchestration Patterns](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns) - Microsoft Azure

### Memory Systems
- [AI Agent Memory: Build Stateful AI Systems](https://redis.io/blog/ai-agent-memory-stateful-systems/) - Redis
- [Context Window Management Strategies](https://www.getmaxim.ai/articles/context-window-management-strategies-for-long-context-ai-agents-and-chatbots) - Maxim
- [Context Engineering for Personalization](https://cookbook.openai.com/examples/agents_sdk/context_personalization) - OpenAI Cookbook

### Intent Recognition
- [Natural Language Understanding (NLU)](https://www.ibm.com/think/topics/natural-language-understanding) - IBM
- [NLP vs NLU: Key Differences](https://www.digitalocean.com/resources/articles/nlp-vs-nlu) - DigitalOcean

### Integration Patterns
- [Claude Integration with n8n](https://n8n.io/integrations/claude/) - n8n
- [How to Connect Claude AI with n8n](https://blog.horizon.dev/connect-claude-ai-with-n8n/) - Horizon
- [Building an AI Telegram Agent with Python and Claude](https://danoncoding.com/building-an-ai-telegram-agent-with-python-and-claude-2f18a0d1a6dc) - Dan On Coding

### AI Frameworks
- [Top AI Agent Frameworks as of February 2026](https://www.shakudo.io/blog/top-9-ai-agent-frameworks) - Shakudo
- [LangChain Documentation](https://python.langchain.com/docs/get_started/introduction) - LangChain
- [AutoGen Multi-Agent Framework](https://microsoft.github.io/autogen/) - Microsoft Research

---

## ✅ Recommended Next Steps

### Immediate (This Session)
1. ✅ Review this architecture document
2. 🔨 Create Redis session memory setup
3. 🔨 Design `ai_memory` table schema

### Next Session
1. Build core CRM API endpoints (Phase 12)
2. Implement tool registry system
3. Create Claude function calling workflow

### Week 1 Goal
- Functional conversational assistant for pipeline queries
- Working memory system (session + persistent)
- 5-10 core tools integrated

---

**Document Version**: 1.0
**Last Updated**: 2026-02-14
**Author**: Claude (AI Architecture Analysis)
**Review Status**: ⏳ Awaiting user feedback
