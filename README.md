# Master Telegram Bot Assistant

AI-powered Telegram assistant for Haven Home Loans CRM with natural language understanding and multi-step task orchestration.

## Overview

This is a conversational AI assistant that integrates with the Haven Home Loans CRM via Telegram. It allows loan officers and admins to:

- **Query pipeline data** ("What's my pipeline value?")
- **Check emails** ("Scan my inbox for new leads")
- **Manage follow-ups** ("Who should I follow up with today?")
- **Analyze loans** ("Are any VA loans at risk?")
- **Schedule tasks** ("Check my calendar and email Joe if I'm free tomorrow at 10")

## Architecture

**Hybrid Intent Recognition**: 80% rule-based (fast), 20% AI-powered (flexible)

**Technology Stack**:
- **n8n**: Workflow orchestration and Telegram integration
- **Claude API**: Natural language understanding and function calling
- **Redis**: Session memory (conversation context)
- **PostgreSQL**: Persistent memory (user preferences, learned patterns)
- **Telegram Bot API**: Conversational interface

## Repository Structure

```
master-telegram-bot-assistant/
├── docs/               # Architecture and implementation docs
├── n8n-workflows/      # n8n workflow JSON files
├── api/                # CRM API integration endpoints
├── memory/             # Memory system (Redis + PostgreSQL schemas)
├── tools/              # Tool definitions for Claude function calling
└── tests/              # Test scenarios and validation
```

## Key Features

- **Natural Language**: No command memorization - just text naturally
- **Context-Aware**: Remembers conversation history and context
- **Multi-Step Tasks**: Orchestrates complex workflows (calendar + email + approval)
- **Tool Calling**: Structured function calls for reliable actions
- **Two-Tier Memory**: Fast session memory + long-term learning
- **Cost-Optimized**: 80/20 rule keeps most queries fast and free

## Documentation

- [Complete Architecture](docs/AI_ASSISTANT_ARCHITECTURE.md) - Full system design
- [Implementation Plan](docs/IMPLEMENTATION_PLAN.md) - Step-by-step roadmap
- [Architecture Summary](docs/ARCHITECTURE_SUMMARY.md) - Visual diagrams & quick reference

## Getting Started

See [IMPLEMENTATION_PLAN.md](docs/IMPLEMENTATION_PLAN.md) for detailed setup instructions.

**Quick Start**:
1. Set up Redis for session memory
2. Create `ai_memory` table in PostgreSQL
3. Deploy n8n Command Router workflow
4. Configure Telegram bot webhook
5. Test with simple queries

## CRM Integration

This bot integrates with the Haven Home Loans CRM:
- **CRM Repo**: https://github.com/harriswat/havenhomeloans-mortgage-crm
- **API Endpoints**: Will be created in Phase 2 of implementation
- **Database**: Shared PostgreSQL instance

## License

Private - Haven Home Loans LLC
