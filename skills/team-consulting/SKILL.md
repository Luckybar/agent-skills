---
name: team-consulting
description: Spins up a persistent team of specialist consultant agents for complex refactoring — each agent owns one reference source and can cross-reference with others via messaging. Use when refactoring involves 4+ reference sources, cross-domain dependencies, or when a single consultant can't hold enough context.
---

# Team-Based Consulting

## Overview

For complex refactoring or rewriting tasks with many reference sources (old code, API specs, DB schemas, business docs, config files), a single consultant subagent may not have enough context to answer cross-domain questions well. This skill uses Claude Code Agent Teams to create persistent specialist agents — each one owns and deeply understands one reference source, and they can cross-reference each other via messaging.

## When to Use

- Refactoring/rewriting with 4+ distinct reference sources
- Sources span different domains (code + database + API spec + business docs)
- Questions frequently require cross-referencing multiple sources
- The legacy system is large enough that one agent can't hold all relevant context

**NOT for:**
- Simple refactoring with 2-3 reference sources (use the single legacy consultant instead)
- Greenfield projects with no legacy to reference
- Tasks where all reference material fits in a single context window

**Decision rule:**

```
How many reference sources?
    │
    ├── 1-3 sources ──→ Single legacy consultant (subagent dispatch)
    │                    Simpler, cheaper, sufficient for most cases
    │
    └── 4+ sources ───→ Team consulting (this skill)
        OR               Better cross-referencing, parallel queries,
        Cross-domain      but ~4x token cost
        dependencies
```

## Team Architecture

```
┌────────────────────────────────────────────────────────────────┐
│  TEAM: consulting-squad                                        │
│                                                                │
│  ┌──────────┐                                                  │
│  │   LEAD   │  Controller — dispatches tasks, routes questions │
│  └────┬─────┘                                                  │
│       │                                                        │
│       ├──────────────┬──────────────┬──────────────┐           │
│       ▼              ▼              ▼              ▼           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │  CODE    │  │  SCHEMA  │  │   API    │  │   DOCS   │      │
│  │ ANALYST  │  │ ANALYST  │  │ ANALYST  │  │ ANALYST  │      │
│  │          │  │          │  │          │  │          │      │
│  │ Reads:   │  │ Reads:   │  │ Reads:   │  │ Reads:   │      │
│  │ old code │  │ DB schema│  │ API specs│  │ business │      │
│  │ git hist │  │ migration│  │ swagger  │  │ reqs/PRD │      │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘      │
│       ▲              ▲              ▲              ▲           │
│       └──────────────┴──────────────┴──────────────┘           │
│                  Can message each other                         │
│               (cross-reference questions)                       │
└────────────────────────────────────────────────────────────────┘
```

## Process

### Step 1: Identify Reference Sources

List all reference sources with their type and domain:

```
TEAM CONSULTING — SOURCE INVENTORY

| Name          | Type          | Path                      | Domain         |
|---------------|---------------|---------------------------|----------------|
| old-code      | codebase      | src/legacy/               | Implementation |
| db-schema     | schema        | db/migrations/            | Data model     |
| api-spec      | documentation | docs/openapi.yaml         | API contracts  |
| business-docs | documentation | docs/requirements/        | Business rules |
| config        | config        | config/                   | Environment    |
| ext-services  | documentation | docs/integrations/        | External APIs  |
```

### Step 2: Design the Team

Map each source (or group of related sources) to a specialist agent:

**Rules for agent assignment:**
- One agent per major domain (code, data, API, business)
- Group closely related sources under one agent (e.g., DB schema + migrations → one schema analyst)
- Don't create more than 5 agents — diminishing returns, coordination overhead grows
- Every agent must have a clear, non-overlapping responsibility

**Agent definition template:**

```
Agent: [name]
Source: [path(s)]
Responsibility: [what questions this agent answers]
Cross-references with: [which other agents it might need to ask]
```

### Step 3: Create the Team

Use TeamCreate to establish the team, then spawn each analyst:

```
1. TeamCreate: "consulting-squad"
2. For each analyst:
   - Spawn via Agent tool with team_name="consulting-squad"
   - Provide: agent name, source paths, responsibility scope
   - Instruct: "You are a [domain] analyst. Your sources are [paths].
     You answer questions about [scope]. Cite file:line for every claim.
     You can message other analysts to cross-reference."
3. Create initial tasks for each analyst:
   - "Familiarize yourself with your sources — read key files, build a mental map"
   - This warm-up step ensures analysts have loaded context before questions arrive
```

### Step 4: Query Protocol

When the implementer (or controller) has a question:

#### Simple query (single domain):
```
Controller → TaskCreate: "Question: How does auth middleware work?"
         → Assign to: code-analyst
         → code-analyst reads old code, answers with evidence
```

#### Cross-domain query:
```
Controller → TaskCreate: "Question: What DB tables does the order API write to, and what are the field constraints?"
         → Assign to: api-analyst (primary)
         → api-analyst reads API spec, identifies endpoints
         → api-analyst → SendMessage to schema-analyst: "What tables do POST /orders and PUT /orders/:id write to?"
         → schema-analyst reads schema, responds with table definitions
         → api-analyst synthesizes both and reports back
```

#### Broad investigation:
```
Controller → TaskCreate: "Investigate: Map the complete user registration flow — API entry, business rules, data stored, config dependencies"
         → Assign to: all analysts (broadcast)
         → Each analyst investigates their domain
         → Lead synthesizes into unified flow map
```

### Step 5: Integration with Build Loop

During `subagent-driven-development`, the team is used as follows:

```
Implementer reports NEEDS_CONTEXT
    │
    ▼
Controller classifies the question
    │
    ├── Single domain? ──→ Route to specific analyst via task
    ├── Cross-domain? ───→ Route to primary analyst, who cross-refs
    └── Broad? ──────────→ Broadcast to all, lead synthesizes
    │
    ▼
Answer fed back to implementer → continue implementation
```

The team stays alive for the entire build phase. Analysts accumulate context across multiple questions — this is the key advantage over ephemeral consultant subagents.

### Step 6: Teardown

After all implementation tasks complete:

1. Ask each analyst for a final summary of their domain knowledge
2. Save summaries to `docs/legacy-analysis/` for future reference
3. Send shutdown_request to each analyst
4. TeamDelete to clean up

## Communication Protocol

### Message Format

Analysts should use this format when messaging each other:

```
FROM: [agent-name]
QUESTION: [specific question]
CONTEXT: [why I'm asking — what I'm trying to answer for the controller]
```

### Response Format

```
FROM: [agent-name]
ANSWER: [direct answer]
EVIDENCE: [source: file:line — what it shows]
CAVEAT: [anything uncertain or that needs human verification]
```

### Rules

1. **Ask specific questions.** "Tell me about the database" is too broad. "What columns does the users table have and which are nullable?" is specific.
2. **Include context for why you're asking.** This helps the other analyst give a relevant answer rather than a generic dump.
3. **Don't relay — investigate.** If code-analyst needs to know about a table, they message schema-analyst directly rather than going through the lead.
4. **Flag disagreements.** If two analysts find conflicting information, they should both report the discrepancy to the lead with evidence.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "I'll just use one consultant for everything, it's simpler" | One consultant with 6 sources will give shallow answers. Specialists with one source each give deep, cited answers. |
| "The team setup takes too long, I'll skip it" | Setup takes 2-3 minutes. Repeatedly dispatching ephemeral consultants that re-read the same files wastes more time over a 10-task build. |
| "I don't need the warm-up step, analysts can read on-demand" | Pre-loading sources means answers come faster and with better cross-file understanding. The warm-up pays for itself on the second question. |
| "5 analysts aren't enough, I need one per file" | More agents = more coordination overhead. Group related sources. If one analyst covers all DB-related files, they'll give better answers than three analysts each seeing one migration file. |
| "I'll keep the team running after the build, might need it later" | Teams are expensive (persistent processes). Tear down after the build and save the summaries. Spin up again if needed. |

## Red Flags

- Creating more than 5 analyst agents (coordination overhead exceeds benefit)
- Analysts answering questions outside their source domain
- Analysts writing code instead of reading and explaining
- Skipping the warm-up step (analysts won't have loaded context)
- Leaving the team running after the build phase ends
- Not saving analyst summaries before teardown (knowledge lost)
- Controller answering questions from memory instead of routing to analysts

## Verification

After team consulting completes, confirm:
- [ ] Each analyst's sources were clearly scoped (no overlapping responsibility)
- [ ] Warm-up step completed (analysts loaded their source context)
- [ ] Questions were routed to the right analyst based on domain
- [ ] Cross-domain questions involved direct analyst-to-analyst communication
- [ ] All answers include source:file:line evidence
- [ ] Discrepancies between sources were flagged
- [ ] Final domain summaries saved to docs/legacy-analysis/
- [ ] Team properly shut down after build completion
