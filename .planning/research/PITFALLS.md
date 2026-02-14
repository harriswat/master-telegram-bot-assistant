# Domain Pitfalls

**Domain:** Production Telegram bot with AI assistant, CRM integration, and n8n orchestration
**Researched:** 2026-02-14
**Overall Confidence:** HIGH (multiple sources, official documentation, real-world incident reports)

---

## Critical Pitfalls

Mistakes that cause rewrites, security incidents, or major production failures.

---

### Pitfall 1: Bot Token Exposure in Source Code or Logs

**What goes wrong:** The Telegram bot token is hardcoded in source files, committed to Git, or leaked through error messages. An attacker who obtains the token gains full control of the bot.

**Why it happens:** Developers hardcode tokens during prototyping. Error handlers log full request URLs (which contain the token). GitHub detected over 39 million leaked secrets in 2024.

**Consequences:** Complete bot takeover -- attacker sends messages as the bot, reads all history.

**Prevention:**
- Store bot token exclusively in Doppler (project's existing secret manager)
- Add `BOT_TOKEN` patterns to `.gitignore` and enable GitHub secret scanning with push protection
- Sanitize all error logs: strip tokens from URLs before logging
- Use `pre-commit` hooks to block commits containing secrets

**Detection:** Search codebase for token strings periodically. Monitor for unexpected bot activity.

---

### Pitfall 2: No Chat ID Whitelisting (Missing Authentication)

**What goes wrong:** The bot accepts messages from anyone who discovers the bot username. Since this handles loan data and pipeline info, any Telegram user could query sensitive business data.

**Why it happens:** Telegram bots are public by default. The Bot API provides no built-in authentication -- it is entirely the developer's responsibility.

**Consequences:** Unauthorized access to CRM data, commission figures, client names.

**Prevention:**
- Whitelist Harris's `chat_id` as the FIRST check in the command router workflow, before ANY processing
- Return no meaningful response to unauthorized users
- Log all unauthorized access attempts for security monitoring
- Store whitelist in Doppler, not hardcoded

**Detection:** Monitor logs for messages from non-whitelisted chat IDs.

---

### Pitfall 3: Webhook Timeout Causing Duplicate Processing

**What goes wrong:** The n8n webhook handler takes too long (AI calls take 2-5s, CRM queries add latency). Telegram re-sends the update, causing duplicate responses and CRM operations.

**Why it happens:** Chaining Claude API + CRM API + Microsoft Graph in a single handler can exceed Telegram's timeout threshold. Once you respond to a webhook, Telegram sends the next update immediately, creating race conditions.

**Consequences:** Duplicate messages, duplicate CRM activities logged, conversation state corruption.

**Prevention:**
- Acknowledge webhook immediately (HTTP 200 within 1-2 seconds) using n8n's "Respond to Webhook" node early in the workflow
- Send typing indicator as the immediate response, then process asynchronously
- Implement idempotency: track processed `update_id` values and skip duplicates
- For multi-step tasks, break into separate workflows with state tracking

**Detection:** Log every webhook with `update_id` and timestamp. Alert on duplicate processing.

---

### Pitfall 4: Claude API Costs Spiraling Out of Control

**What goes wrong:** Every message triggers a full Claude API call with entire conversation history, tool definitions, and system prompt. At $15/M output tokens for Sonnet, a chatty user could run $15-30/day.

**Why it happens:** No rule-based routing for simple queries. Full context sent every time. No model tiering.

**Consequences:** Monthly AI costs of $200-400+ instead of budgeted $20-40.

**Prevention:**
- Route 80% of queries through rule engine at $0 AI cost
- Use Haiku 4.5 ($1/$5 per MTok) as primary model, not Sonnet ($3/$15)
- Cache tool definitions via Anthropic prompt caching (up to 71% savings)
- Limit conversation history to last 10 messages
- Track costs daily and set alerting thresholds

**Detection:** Log token usage per Claude API call. Dashboard or weekly cost report.

---

### Pitfall 5: Direct Database Access from Bot (Bypassing API Layer)

**What goes wrong:** n8n workflows query PostgreSQL directly using n8n's built-in Postgres node instead of going through CRM API endpoints. This creates tight coupling, bypasses business logic, and skips audit logging.

**Why it happens:** n8n has a built-in Postgres node that makes direct queries trivially easy. The CRM API endpoints are not yet built, creating temptation to "just query directly for now."

**Consequences:** Schema changes break the bot silently. Role-based access bypassed. No audit trail. Two separate query patterns to maintain.

**Prevention:**
- Build `/api/assistant/*` endpoints FIRST
- All bot data access goes through these HTTP endpoints
- Never use n8n's PostgreSQL node for CRM data
- If a quick-hack direct query is needed during development, flag it with `TODO: Replace with API call`

**Detection:** Search n8n workflows for PostgreSQL nodes connecting to the CRM database.

---

## Moderate Pitfalls

Mistakes that degrade user experience or create maintenance burden.

---

### Pitfall 6: Conversation State Lost on Restart or Timeout

**What goes wrong:** The user is halfway through approving an email draft. n8n restarts (deployment, crash) or session times out. User sends "yes" but bot has no context.

**Why it happens:** State stored only in Valkey is lost if TTL expires or Valkey is restarted. Critical pending actions need durability.

**Prevention:**
- Persist critical pending actions to PostgreSQL (survive Valkey failures)
- Include unique `action_id` in inline keyboard callbacks for lookup without session context
- Handle orphaned approvals: "That action has expired. Want me to set it up again?"
- Set 15-minute TTL for approvals (not 1 hour -- stale approvals are dangerous)

**Detection:** Track orphaned callback events. Monitor Valkey key expiration rates.

---

### Pitfall 7: Not Answering Callback Queries (Inline Button Clicks)

**What goes wrong:** User taps inline button. Bot processes action but does not call `answerCallbackQuery`. 15-second loading spinner appears.

**Why it happens:** Developers focus on action logic, forget Telegram requires explicit acknowledgment.

**Consequences:** 15-second spinner. Users tap repeatedly, causing duplicates.

**Prevention:**
- "Telegram: Answer Callback Query" as FIRST step in every callback handler
- Use `show_alert: true` for important confirmations
- Answer immediately with "Processing..." then follow up

**Detection:** Test every inline button. No spinners should persist.

---

### Pitfall 8: AI Hallucinating CRM Data

**What goes wrong:** Claude invents loan amounts, client names, or closing dates not in the CRM. User makes business decisions on fabricated data.

**Why it happens:** Tool call fails, error not handled properly, Claude answers from general knowledge.

**Prevention:**
- System prompt: "NEVER guess numbers, dates, or statuses. If a tool fails, say 'I could not retrieve that.'"
- Validate all structured data in n8n before passing to Claude
- Include data source: "According to your CRM, Kevin's loan is at $500k"
- For financial data, use exact API numbers, not Claude's paraphrase

**Detection:** Compare bot responses against CRM data. Track tool success/failure rates.

---

### Pitfall 9: Telegram API Rate Limits

**What goes wrong:** Bot sends too many messages, hits 30/second global limit. Returns 429. Persistent violations cause bans.

**Prevention:**
- Per-chat rate limiting: max 1 message/second
- Exponential backoff on 429, respect `Retry-After` header
- Never retry 403/400 -- only 429 and 5xx

---

### Pitfall 10: Monolithic n8n Workflow

**What goes wrong:** All bot logic in one massive workflow. Impossible to debug, test, or modify.

**Prevention:**
- Router does ONLY: authenticate, classify, dispatch, respond
- Each domain is a separate sub-workflow
- Execute Sub-Workflow node for separation
- Keep workflows under 20-25 nodes

---

### Pitfall 11: No User Feedback During Long Operations

**What goes wrong:** 3-5 second silence while AI processes. User thinks bot is broken.

**Prevention:**
- `sendChatAction("typing")` immediately on every message
- Quick acknowledgment for >2s operations
- Progress updates for multi-step tasks
- Edit acknowledgment with final result

---

### Pitfall 12: n8n Redis Chat Memory Pollution (Known Bug)

**What goes wrong:** n8n AI Agent v3.0 saves full intermediate tool outputs into Redis Chat Memory, even when `returnIntermediateSteps = false`. Inflates tokens and costs on subsequent messages.

**Why it happens:** Known bug (GitHub #22112).

**Prevention:**
- Post-processing Code node that strips large tool results from session before Valkey save
- Monitor session sizes
- Max session size check (if > 5000 chars, trim oldest messages)

**Source:** [n8n GitHub Issue #22112](https://github.com/n8n-io/n8n/issues/22112)

---

## Minor Pitfalls

### Pitfall 13: Sending New Messages Instead of Editing
Use `editMessageText` to update previous messages. Keep chat clean.

### Pitfall 14: Missing /start, /help, /cancel Commands
Implement and register with @BotFather. Users need discovery and escape mechanisms.

### Pitfall 15: Stale Webhook Updates After Restart
Check `date` field. Skip messages older than 120 seconds.

### Pitfall 16: Exposing Internal Error Details
Centralized error handler. Never expose stack traces, connection strings, or tokens.

### Pitfall 17: Overcomplicating System Prompt
Keep base prompt under 500 tokens. Load tools conditionally. Summarize history aggressively.

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| **Webhook Setup** | Timeout / duplicate processing (#3) | Acknowledge immediately, offload heavy work |
| **Chat ID Auth** | No whitelist (#2) | Enforce as first middleware check |
| **CRM API Build** | Direct DB access temptation (#5) | Build API endpoints first |
| **Claude Integration** | Cost spiral (#4) | Rule-based 80%, model tiering |
| **Claude Integration** | Hallucinated data (#8) | System prompt guardrails |
| **Memory System** | State lost during approvals (#6) | Persist to PostgreSQL |
| **Memory System** | Memory pollution (#12) | Post-processing sanitizer |
| **Inline Keyboards** | Missing answerCallbackQuery (#7) | First step in callback handler |
| **Workflow Design** | Monolithic workflow (#10) | Split by domain |
| **UX** | No feedback during processing (#11) | Typing indicator + progress |
| **Production** | Token exposure (#1) | Doppler only |

---

## Sources

### Official Documentation
- [Telegram Bot API](https://core.telegram.org/bots/api)
- [Telegram Bot Features](https://core.telegram.org/bots/features)
- [Telegram Bots FAQ](https://core.telegram.org/bots/faq)
- [n8n Telegram Trigger Common Issues](https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.telegramtrigger/common-issues/)
- [n8n Handling API Rate Limits](https://docs.n8n.io/integrations/builtin/rate-limits/)

### Security
- [GitGuardian: Telegram Bot Token Remediation](https://www.gitguardian.com/remediation/telegram-bot-token)
- [BAZU: How to Secure a Telegram Bot](https://bazucompany.com/blog/how-to-secure-a-telegram-bot-best-practices/)

### Cost and Performance
- [Claude Cost Optimization with Telegram](https://blog.jasoncochran.io/the-complete-guide-to-claude-cost-optimization-with-clawdbot-and-telegram/)
- [Anthropic Pricing](https://www.anthropic.com/pricing)

### Known Issues
- [n8n GitHub #22112: AI Agent Memory Pollution](https://github.com/n8n-io/n8n/issues/22112)
- [n8n Community: Long Prompts with Redis Memory](https://community.n8n.io/t/issue-with-long-prompt-length-due-to-full-conversation-history-in-n8n-memory-buffer-redis/100986)

### User Experience
- [10 Best UX Practices for Telegram Bots](https://medium.com/@bsideeffect/10-best-ux-practices-for-telegram-bots-79ffed24b6de)
