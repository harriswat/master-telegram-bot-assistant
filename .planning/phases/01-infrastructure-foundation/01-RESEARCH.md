# Phase 1: Infrastructure & Foundation - Research

**Researched:** 2026-02-14
**Domain:** DigitalOcean Managed Valkey, Next.js API route handlers, n8n Telegram Trigger, PostgreSQL schema extension, Prisma migrations
**Confidence:** HIGH

## Summary

Phase 1 establishes the foundational infrastructure for the Telegram AI assistant: a Valkey session store, CRM API endpoints for the bot to consume, a basic Telegram echo workflow proving webhook connectivity, chat ID whitelisting for security, and a `bot_logs` table for compliance logging.

The existing CRM codebase (Next.js 16.1.6 with Prisma 6.19.2 on DigitalOcean Managed PostgreSQL 17) provides a solid foundation. The CRM already has working API routes (`/api/webhooks/*`), a Prisma singleton, server actions with role-based access, and a proxy middleware that exempts `/api/webhooks` from authentication. Phase 1 adds a parallel `/api/assistant/*` route namespace secured by API key (not session cookies), a new `bot_logs` Prisma model, a Valkey instance on DigitalOcean, and a single n8n workflow that receives Telegram messages and echoes them back.

The research domains are all well-established technologies with verified documentation. The main risk is Valkey TLS configuration for n8n's Redis credential system -- DigitalOcean requires TLS, and n8n's Redis credential has an SSL toggle that must be enabled. A pre-existing Telegram Command Router workflow exists in the codebase (`integrations/telegram-ai-assistant/workflows/json/1-telegram-command-router.json`) but it lacks whitelisting, logging, and is not deployed to the live n8n instance. Phase 1 will build a simpler echo-only workflow from scratch.

**Primary recommendation:** Build CRM API endpoints first (they are the data layer everything else depends on), then Valkey provisioning, then the Telegram echo workflow with whitelisting and logging. Do not skip the API key authentication on assistant endpoints.

## Standard Stack

### Core (already in project)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Next.js | 16.1.6 | API route handlers for `/api/assistant/*` | Already the project framework. Route handlers support Web Request/Response APIs. |
| Prisma | 6.19.2 | ORM for `bot_logs` table + existing CRM queries | Already configured with DO Managed PostgreSQL. Schema extension via `prisma migrate dev`. |
| TypeScript | 5.9.3 | Type safety for API routes and schemas | Already configured project-wide. |
| Zod | 4.3.6 | Request/response validation for API endpoints | Already installed. Use for API key validation and query param parsing. |

### New Infrastructure

| Technology | Version | Purpose | Why Standard |
|------------|---------|---------|--------------|
| DigitalOcean Managed Valkey | 8.0 | Session memory store for conversation context | Drop-in Redis replacement. $15/mo. Same VPC as existing infrastructure. TLS encrypted. n8n Redis nodes work unchanged. |
| n8n Telegram Trigger | v1.1 (node version) | Receive Telegram messages via webhook | Already have n8n deployed. Telegram Trigger is the standard entry point. Single webhook per bot constraint. |
| n8n Telegram Send Message | v1.2 (node version) | Reply to Telegram messages | Standard n8n Telegram node for responses. Supports Markdown formatting. |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| date-fns | 4.1.0 | Date formatting in API responses | Already installed. Use for human-readable date strings in loan/lead responses. |
| Doppler | CLI v3.75.2 | Secret management for new secrets (VALKEY_URL, ASSISTANT_API_KEY, TELEGRAM_BOT_TOKEN, TELEGRAM_CHAT_ID) | Already configured as the project's secret manager. All new secrets go here. |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Valkey (DO Managed) | Upstash Redis (serverless) | Upstash is cheaper at very low volume but adds another vendor. DO keeps everything on one platform. |
| API key auth for assistant routes | Session-based auth (existing Better Auth) | n8n cannot maintain browser sessions. API key is the correct pattern for machine-to-machine communication. |
| New `/api/assistant/*` routes | Reuse existing server actions | Server Actions use `"use server"` directive and `headers()` for session auth -- incompatible with external API key auth. Separate routes are correct. |
| Prisma `bot_logs` model | Direct SQL table creation | Prisma migration keeps schema in sync with code. Consistent with existing patterns. |

**No new npm packages required for Phase 1.** All dependencies are already installed.

## Architecture Patterns

### Recommended Project Structure (new files for Phase 1)

```
src/
  app/
    api/
      assistant/
        _lib/
          auth.ts           # validateAssistantKey() helper
          serialize.ts      # Decimal-to-number serializers for API responses
        loans/
          active/route.ts   # GET - Active pipeline overview
          [id]/route.ts     # GET - Single loan details
          search/route.ts   # GET - Search loans by borrower name
        leads/
          recent/route.ts   # GET - Recent leads
        partners/
          top/route.ts      # GET - Top partners by performance
prisma/
  schema.prisma             # Add BotLog model
  migrations/
    YYYYMMDD_add_bot_logs/  # Migration for bot_logs table
```

### Pattern 1: API Key Authentication for Assistant Endpoints

**What:** A shared helper that validates the `X-Assistant-Key` header on every `/api/assistant/*` request.
**When to use:** Every assistant API endpoint must call this before processing.
**Why:** n8n cannot maintain browser sessions. API key is the standard pattern for machine-to-machine auth. The key is stored in Doppler and set in n8n's HTTP Request Tool header configuration.

```typescript
// Source: Verified pattern from Next.js 16 Route Handler docs + existing webhook auth pattern
// src/app/api/assistant/_lib/auth.ts

import { NextResponse } from 'next/server';

export function validateAssistantKey(request: Request): boolean {
  const key = request.headers.get('X-Assistant-Key');
  return key === process.env.ASSISTANT_API_KEY;
}

export function unauthorizedResponse() {
  return NextResponse.json(
    { error: 'Unauthorized' },
    { status: 401 }
  );
}
```

**Proxy middleware update needed:** The existing `proxy.ts` exempts `/api/webhooks` from session auth. Must also exempt `/api/assistant` so that API key auth (not session cookies) is used.

```typescript
// In src/proxy.ts, update publicRoutes:
const publicRoutes = ["/login", "/api/auth", "/api/webhooks", "/api/assistant"];
```

### Pattern 2: Decimal Serialization in API Responses

**What:** Convert Prisma Decimal fields to plain numbers before returning JSON.
**When to use:** Every API endpoint returning loan/lead data with monetary fields.
**Why:** Prisma Decimal objects cannot be serialized to JSON. The existing codebase already uses `serializeLoan()` and `serializeLead()` helpers in server actions. Create a shared version for API routes.

```typescript
// Source: Existing pattern from src/app/(dashboard)/loans/utils.ts
// src/app/api/assistant/_lib/serialize.ts

export function serializeDecimal(value: any): number | null {
  if (value === null || value === undefined) return null;
  if (typeof value === 'number') return value;
  if (typeof value.toNumber === 'function') return value.toNumber();
  return Number(value);
}

export function serializeLoan(loan: any) {
  return {
    ...loan,
    loanAmount: serializeDecimal(loan.loanAmount),
    purchasePrice: serializeDecimal(loan.purchasePrice),
    ltv: serializeDecimal(loan.ltv),
    rate: serializeDecimal(loan.rate),
    compPercent: serializeDecimal(loan.compPercent),
    estimatedCompAmount: serializeDecimal(loan.estimatedCompAmount),
    actualCompAmount: serializeDecimal(loan.actualCompAmount),
  };
}
```

### Pattern 3: Next.js 16 Route Handler Pattern

**What:** Standard GET/POST route handler pattern for the App Router.
**When to use:** All `/api/assistant/*` endpoints.

```typescript
// Source: Next.js 16.1.6 official docs (https://nextjs.org/docs/app/getting-started/route-handlers)
// src/app/api/assistant/loans/active/route.ts

import { NextRequest, NextResponse } from 'next/server';
import { db } from '@/lib/db';
import { validateAssistantKey, unauthorizedResponse } from '../../_lib/auth';
import { serializeLoan } from '../../_lib/serialize';

export const dynamic = 'force-dynamic';

export async function GET(request: NextRequest) {
  if (!validateAssistantKey(request)) {
    return unauthorizedResponse();
  }

  const { searchParams } = request.nextUrl;
  const status = searchParams.get('status');
  const closingSoon = searchParams.get('closing_soon') === 'true';
  const lockExpiring = searchParams.get('lock_expiring') === 'true';

  const where: any = {
    status: { notIn: ['FUNDED', 'CLOSED', 'DENIED', 'WITHDRAWN'] },
  };

  if (status) {
    where.status = status;
  }

  const loans = await db.loan.findMany({
    where,
    include: {
      borrower: { select: { firstName: true, lastName: true, email: true, phone: true } },
      loanOfficer: { select: { id: true, name: true } },
      referralPartner: { select: { firstName: true, lastName: true, companyName: true } },
    },
    orderBy: { closingDate: 'asc' },
  });

  return NextResponse.json({
    count: loans.length,
    loans: loans.map(serializeLoan),
  });
}
```

### Pattern 4: n8n Telegram Echo Workflow (Phase 1 minimal)

**What:** Single workflow with Telegram Trigger -> Chat ID Check -> Log -> Echo Reply.
**When to use:** Phase 1 only. This is replaced with the full Command Router in Phase 2.

```
Workflow: "Telegram AI Assistant - Echo (Phase 1)"
Nodes:
  1. [Telegram Trigger] - Webhook mode, listens for "message" events
  2. [Code: Check Whitelist] - Compare chat_id against allowed list
     - If not whitelisted: [Telegram: Send "Unauthorized"] -> [Stop]
  3. [HTTP Request: Log to CRM] - POST /api/assistant/bot-log with message data
  4. [Telegram: Send Reply] - Echo "Received: {message_text}"
```

**Critical:** The Telegram Trigger node registers a webhook with Telegram's servers. Only ONE workflow can use the Telegram Trigger for a given bot. Activating a second workflow will overwrite the webhook.

### Pattern 5: Bot Log Entry via API

**What:** Every incoming Telegram message is logged via an API call to the CRM.
**When to use:** First step after whitelist check in every workflow.

```typescript
// Source: Phase requirements (LOG-01) + existing WebhookLog pattern in codebase
// src/app/api/assistant/bot-log/route.ts

import { NextRequest, NextResponse } from 'next/server';
import { db } from '@/lib/db';
import { validateAssistantKey, unauthorizedResponse } from '../_lib/auth';

export async function POST(request: NextRequest) {
  if (!validateAssistantKey(request)) {
    return unauthorizedResponse();
  }

  const body = await request.json();

  const log = await db.botLog.create({
    data: {
      chatId: String(body.chat_id),
      messageText: body.message_text || '',
      routeTaken: body.route_taken || 'echo',
      responseText: body.response_text || '',
      processingTimeMs: body.processing_time_ms || 0,
      metadata: body.metadata || {},
    },
  });

  return NextResponse.json({ id: log.id, status: 'logged' });
}
```

### Anti-Patterns to Avoid

- **Direct database access from n8n:** Never use n8n's PostgreSQL node to query the CRM database directly. All data access must go through `/api/assistant/*` endpoints. This preserves business logic, audit logging, and schema independence.
- **Session-based auth on assistant routes:** n8n cannot maintain browser sessions. Use API key auth exclusively for assistant endpoints.
- **Multiple Telegram Trigger workflows:** Telegram allows ONE webhook per bot. Multiple workflows with Telegram Trigger will overwrite each other.
- **Hardcoding bot token or API keys:** All secrets in Doppler. Never in code, workflow JSON, or environment files.
- **Skipping the proxy middleware update:** If `/api/assistant` is not added to `publicRoutes` in `proxy.ts`, the session cookie check will reject all n8n requests with a redirect to `/login`.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Session memory store | In-memory Map or PostgreSQL polling | DigitalOcean Managed Valkey ($15/mo) | Sub-1ms latency, TTL support, n8n Redis Chat Memory node works directly, managed failover and encryption |
| Decimal serialization | Manual `Number()` casting | Shared `serializeDecimal()` helper | Prisma Decimal has `.toNumber()` method. Consistent pattern already used in codebase. |
| Telegram webhook management | Manual `setWebhook` API calls | n8n Telegram Trigger node | Automatically registers/deregisters webhooks when workflow is activated/deactivated |
| API key management | Custom key generation/rotation | Doppler secret + simple header check | Single-user system. A random UUID as API key is sufficient. Rotation via Doppler. |
| TLS for Valkey | Self-managed certificates | DigitalOcean managed encryption | DO handles TLS termination, certificate rotation, and encryption at rest |

**Key insight:** Phase 1 is about infrastructure plumbing, not application logic. Every component has a well-tested managed solution. Custom code should be limited to the API route handlers and the Prisma schema extension.

## Common Pitfalls

### Pitfall 1: Proxy Middleware Blocking Assistant API Requests

**What goes wrong:** n8n sends requests to `/api/assistant/*` but gets redirected to `/login` because the proxy middleware checks for session cookies.
**Why it happens:** The existing `proxy.ts` only exempts `/login`, `/api/auth`, and `/api/webhooks` from session checks.
**How to avoid:** Add `/api/assistant` to the `publicRoutes` array in `proxy.ts` BEFORE building any endpoints.
**Warning signs:** n8n HTTP Request nodes return 302 redirects or HTML login page content instead of JSON.

### Pitfall 2: Valkey TLS Connection Failure from n8n

**What goes wrong:** n8n's Redis Chat Memory or Redis node cannot connect to DigitalOcean Managed Valkey.
**Why it happens:** DigitalOcean requires TLS for all managed database connections. n8n's Redis credential has an SSL toggle that defaults to OFF. Also, DO uses port 25061 for TLS (not the default 6379).
**How to avoid:** In n8n Redis credential configuration: (1) set Host to the DO hostname, (2) set Port to 25061, (3) enter the password from DO connection details, (4) toggle SSL ON.
**Warning signs:** Connection timeout or "ECONNREFUSED" errors in n8n execution logs.

### Pitfall 3: Telegram Webhook Overwrite

**What goes wrong:** Activating a test workflow with a Telegram Trigger overwrites the production webhook. The live bot stops receiving messages.
**Why it happens:** Telegram allows only ONE webhook per bot token. Each n8n Telegram Trigger node calls `setWebhook` on activation.
**How to avoid:** Use only ONE workflow with a Telegram Trigger for the bot. For testing, use a separate bot token (create via @BotFather).
**Warning signs:** Bot stops responding in production after testing a workflow.

### Pitfall 4: Prisma Decimal Serialization Errors in API Routes

**What goes wrong:** API route returns `TypeError: Do not know how to serialize a BigInt` or empty objects where numbers should be.
**Why it happens:** Prisma Decimal type is not JSON-serializable. The existing server actions handle this via `serializeLoan()`, but new API routes need the same treatment.
**How to avoid:** Use the shared `serializeDecimal()` helper on all monetary fields before `NextResponse.json()`.
**Warning signs:** API returns `{}` for Decimal fields, or TypeScript compilation errors about Decimal types.

### Pitfall 5: n8n HTTP Request Node to CRM Returns HTML Instead of JSON

**What goes wrong:** n8n makes a GET request to `/api/assistant/loans/active` and receives HTML (the login page) instead of JSON.
**Why it happens:** Either the proxy middleware is blocking (Pitfall 1) or the `X-Assistant-Key` header is missing/wrong.
**How to avoid:** (1) Verify proxy exemption. (2) In n8n HTTP Request Tool, add header: `X-Assistant-Key` = `{{$credentials.assistantApiKey}}`. (3) Test the endpoint with curl first.
**Warning signs:** n8n error: "Expected JSON but received text/html".

### Pitfall 6: Stale Telegram Messages After Bot Restart

**What goes wrong:** When the n8n workflow is reactivated, it processes old messages that were queued while the webhook was down.
**Why it happens:** Telegram queues updates when the webhook URL returns errors. When the webhook comes back, all queued updates are delivered.
**How to avoid:** Add a timestamp check: skip messages older than 120 seconds from the current time. Compare `message.date` (Unix timestamp) against `Math.floor(Date.now() / 1000) - 120`.
**Warning signs:** Bot sends multiple old responses when workflow is first activated.

### Pitfall 7: Missing `force-dynamic` on API Routes

**What goes wrong:** API route returns cached/stale data that does not reflect recent CRM changes.
**Why it happens:** Next.js 16 aggressively caches route handler responses by default.
**How to avoid:** Add `export const dynamic = 'force-dynamic';` to every assistant API route file.
**Warning signs:** Loan statuses or lead counts don't update in bot responses even though CRM shows changes.

## Code Examples

### BotLog Prisma Model

```prisma
// Source: Phase requirements (LOG-01) + existing WebhookLog pattern in schema
// Add to prisma/schema.prisma

model BotLog {
  id               String   @id @default(cuid())
  chatId           String   // Telegram chat_id
  messageText      String   @db.Text
  routeTaken       String   // "echo", "rule", "haiku", "sonnet"
  responseText     String?  @db.Text
  toolsCalled      Json?    // Array of tool invocations
  processingTimeMs Int      @default(0)
  tokensUsed       Int?     // Claude API tokens (when applicable)
  error            String?  @db.Text
  metadata         Json?    // Additional context
  createdAt        DateTime @default(now())

  @@index([chatId])
  @@index([createdAt])
  @@index([routeTaken])
  @@map("bot_logs")
}
```

### Loan Search by Borrower Name

```typescript
// Source: Phase requirements (CRM-01) + existing Prisma query patterns
// src/app/api/assistant/loans/search/route.ts

import { NextRequest, NextResponse } from 'next/server';
import { db } from '@/lib/db';
import { validateAssistantKey, unauthorizedResponse } from '../../_lib/auth';
import { serializeLoan } from '../../_lib/serialize';

export const dynamic = 'force-dynamic';

export async function GET(request: NextRequest) {
  if (!validateAssistantKey(request)) {
    return unauthorizedResponse();
  }

  const query = request.nextUrl.searchParams.get('q');
  if (!query || query.length < 2) {
    return NextResponse.json({ error: 'Query must be at least 2 characters' }, { status: 400 });
  }

  const loans = await db.loan.findMany({
    where: {
      borrower: {
        OR: [
          { firstName: { contains: query, mode: 'insensitive' } },
          { lastName: { contains: query, mode: 'insensitive' } },
        ],
      },
    },
    include: {
      borrower: { select: { firstName: true, lastName: true, email: true, phone: true } },
      loanOfficer: { select: { id: true, name: true } },
    },
    orderBy: { updatedAt: 'desc' },
    take: 10,
  });

  return NextResponse.json({
    query,
    count: loans.length,
    loans: loans.map(serializeLoan),
  });
}
```

### n8n Whitelist Check (Code Node)

```javascript
// Source: Phase requirements (security) + Telegram Bot API best practices
// n8n Code node: "Check Whitelist"

const chatId = String($input.item.json.message.chat.id);
const allowedChatIds = $env.TELEGRAM_ALLOWED_CHAT_IDS.split(',').map(id => id.trim());

const isAllowed = allowedChatIds.includes(chatId);
const messageDate = $input.item.json.message.date;
const now = Math.floor(Date.now() / 1000);
const isStale = (now - messageDate) > 120; // Skip messages older than 2 minutes

return {
  json: {
    chatId,
    isAllowed,
    isStale,
    messageText: $input.item.json.message.text || '',
    messageId: $input.item.json.message.message_id,
    userName: $input.item.json.message.from.first_name || 'Unknown',
    timestamp: messageDate,
  }
};
```

### Doppler Secret Setup Commands

```bash
# Generate a random API key for assistant endpoints
ASSISTANT_KEY=$(openssl rand -hex 32)

# Add new secrets to Doppler
doppler secrets set ASSISTANT_API_KEY="$ASSISTANT_KEY" --project mortgage-crm --config dev
doppler secrets set VALKEY_URL="rediss://default:PASSWORD@HOST:25061" --project mortgage-crm --config dev
doppler secrets set TELEGRAM_ALLOWED_CHAT_IDS="HARRIS_CHAT_ID" --project mortgage-crm --config dev

# Verify secrets are set
doppler secrets get ASSISTANT_API_KEY VALKEY_URL TELEGRAM_ALLOWED_CHAT_IDS --project mortgage-crm --config dev
```

### Testing an Assistant Endpoint with curl

```bash
# Test active loans endpoint
curl -H "X-Assistant-Key: YOUR_KEY_HERE" \
  http://localhost:3000/api/assistant/loans/active

# Test loan search
curl -H "X-Assistant-Key: YOUR_KEY_HERE" \
  "http://localhost:3000/api/assistant/loans/search?q=Kevin"

# Test bot log creation
curl -X POST -H "X-Assistant-Key: YOUR_KEY_HERE" \
  -H "Content-Type: application/json" \
  -d '{"chat_id":"12345","message_text":"test","route_taken":"echo"}' \
  http://localhost:3000/api/assistant/bot-log
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `middleware.ts` | `proxy.ts` | Next.js 16 (2026) | File renamed but API identical. This project still uses `middleware.ts` importing `proxy.ts` -- both work. |
| DigitalOcean Managed Redis | DigitalOcean Managed Valkey | June 2025 | Redis discontinued. Valkey is 100% API-compatible drop-in. n8n Redis nodes work unchanged. |
| n8n OpenAI Function Agent | n8n Tools Agent | n8n v1.82 (2025) | All AI Agent nodes now default to Tools Agent pattern. Old Function/ReAct agents are deprecated. |
| Redis port 6379 | Valkey TLS port 25061 | DO Managed Valkey | Must use TLS port and `rediss://` protocol for DigitalOcean managed instances. |
| Prisma 7.x | Prisma 6.19.2 | Project decision (2026-01-31) | Prisma 7.x has breaking changes. Project pinned to 6.19.2. Do not upgrade. |

**Deprecated/outdated:**
- **DigitalOcean Managed Caching (Redis):** Discontinued June 30, 2025. Replaced by Managed Valkey.
- **n8n Function Agent / ReAct Agent:** Deprecated since v1.82. Use Tools Agent exclusively.
- **Redis 7.2:** Reached EOL February 2026. Valkey 8.0 is the current version.

## Open Questions

1. **Valkey + n8n App Platform Networking**
   - What we know: Both n8n and PostgreSQL are on DigitalOcean. Valkey will also be on DO.
   - What's unclear: Whether n8n App Platform can reach DO Managed Valkey via private networking (same VPC) or if public access is required.
   - Recommendation: Create Valkey in NYC3 datacenter (same as PostgreSQL). If private networking does not work, enable "Allow trusted sources" in Valkey firewall rules and add n8n's outbound IP.

2. **Telegram Bot Token -- Is @Bigterrys_bot Already Configured?**
   - What we know: The phase description references `@Bigterrys_bot`. Setup scripts exist in `integrations/telegram-ai-assistant/scripts/setup/`.
   - What's unclear: Whether the bot token and Harris's chat ID are already in Doppler.
   - Recommendation: Check Doppler secrets first. If not present, run the setup script or configure manually.

3. **n8n Redis Credential TLS Toggle -- Exact Behavior**
   - What we know: n8n Redis credential has Host, Port, Password, Database, and SSL toggle fields.
   - What's unclear: Whether the SSL toggle correctly handles `rediss://` protocol internally, or if additional configuration is needed.
   - Recommendation: Test the connection immediately after creating the n8n credential. If SSL toggle alone does not work, try entering the full `rediss://` URL in the host field as a workaround.

4. **Production Deployment of API Routes**
   - What we know: CRM is deployed on DO App Platform at `https://walrus-app-8dgas.ondigitalocean.app`.
   - What's unclear: Whether new API routes are auto-deployed on push to main, or if additional build configuration is needed.
   - Recommendation: Verify deployment pipeline. The existing `build` script (`prisma generate && next build`) should pick up new routes automatically.

## Sources

### Primary (HIGH confidence)
- [DigitalOcean Managed Valkey Documentation](https://docs.digitalocean.com/products/databases/valkey/) -- Pricing ($15/mo starting), features, migration from Redis
- [DigitalOcean Valkey Pricing](https://docs.digitalocean.com/products/databases/valkey/details/pricing/) -- Single node: 1 GiB RAM, $15/mo
- [DigitalOcean Valkey Connection Guide](https://docs.digitalocean.com/products/databases/valkey/how-to/connect/) -- Connection string format, TLS requirement, port 25061
- [n8n Redis Credentials Documentation](https://docs.n8n.io/integrations/builtin/credentials/redis/) -- Host, Port, Password, Database, SSL toggle
- [n8n Telegram Trigger Documentation](https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.telegramtrigger/) -- Webhook mode, event types, common issues
- [n8n Telegram Trigger Common Issues](https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.telegramtrigger/common-issues/) -- Single webhook limitation confirmed
- [Next.js 16 Route Handlers](https://nextjs.org/docs/app/getting-started/route-handlers) -- GET/POST patterns, dynamic exports
- [Next.js 16 Proxy (formerly Middleware)](https://nextjs.org/docs/app/api-reference/file-conventions/proxy) -- Proxy convention, matcher patterns
- Existing codebase: `src/proxy.ts`, `src/app/api/webhooks/arive/loan-created/route.ts`, `prisma/schema.prisma` -- Verified patterns for auth, API routes, schema

### Secondary (MEDIUM confidence)
- [n8n Redis Chat Memory Node](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.memoryredischat/) -- Valkey compatibility confirmed by Redis protocol compatibility
- [Telegram Bot API](https://core.telegram.org/bots/api) -- sendMessage, sendChatAction, getUpdates, webhook registration
- [n8n Community: Telegram Trigger Webhook Issues](https://community.n8n.io/t/problems-with-telegram-webhook/113560) -- HTTPS requirement, WEBHOOK_URL env var
- [n8n Community: Redis/Valkey Connection](https://community.n8n.io/t/issue-connecting-n8n-to-aws-elasticache-redis-valkey-serverless/225993) -- TLS configuration patterns

### Tertiary (LOW confidence -- verify before implementing)
- n8n Redis credential SSL toggle behavior with DigitalOcean Managed Valkey -- needs live testing
- n8n App Platform to DO Managed Valkey private networking -- needs verification during provisioning

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- All technologies already in use or verified with official docs
- Architecture: HIGH -- API route pattern directly mirrors existing webhook routes in codebase
- Pitfalls: HIGH -- Each pitfall verified against codebase patterns or official documentation
- Valkey connectivity: MEDIUM -- TLS configuration needs live testing; protocol compatibility is confirmed

**Research date:** 2026-02-14
**Valid until:** 2026-03-14 (30 days -- all components are stable, no fast-moving dependencies)
