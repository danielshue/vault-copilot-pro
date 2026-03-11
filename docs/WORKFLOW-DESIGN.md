# Vault Copilot — Agentic Workflow Design

This document describes every workflow pattern available in Vault Copilot, how to configure each one, and when to use it. All workflows are defined declaratively using Markdown files in your vault — no code required.

---

## Table of Contents

1. [Workflow Patterns at a Glance](#workflow-patterns-at-a-glance)
2. [Single Agent + Tools](#1-single-agent--tools)
3. [Sequential Pipeline](#2-sequential-pipeline)
4. [Concurrent / Parallel Execution](#3-concurrent--parallel-execution)
5. [Agent Handoff](#4-agent-handoff)
6. [Manager / Planner](#5-manager--planner)
7. [Group Discussion](#6-group-discussion)
8. [Human-in-the-Loop](#7-human-in-the-loop)
9. [Guardrails](#8-guardrails)
10. [Durable / Long-Running Workflows](#9-durable--long-running-workflows)
11. [Session-Based Assistant](#10-session-based-assistant)
12. [Automation Engine](#automation-engine)
13. [Configuration Reference](#configuration-reference)

---

## Workflow Patterns at a Glance

| Pattern | Good For | How to Invoke |
|---------|----------|---------------|
| Single agent + tools | Note lookup, drafting, task execution | Chat with any agent |
| Sequential pipeline | Reports, research pipelines, document generation | `run_pipeline` tool or `run-pipeline` automation action |
| Concurrent / parallel | Multi-source investigation, broad gathering | `run_parallel_agents` tool or `parallel` automation action |
| Agent handoff | Multi-domain assistants, specialist delegation | `handoff_to_agent` tool or handoff buttons in chat |
| Manager / planner | Open-ended tasks, multi-step work with uncertainty | `create_plan` tool |
| Group discussion | Brainstorming, design reviews, tradeoff analysis | `group_discussion` tool |
| Human-in-the-loop | High-risk actions, approvals, external comms | Tool approval settings, `approval-gate` automation action |
| Guardrails | Policy enforcement, safety, PII prevention | `.guardrail.md` files or settings |
| Durable / long-running | Approvals, multi-day investigations, external waits | `wait-for`, `wait-duration`, `approval-gate` actions |
| Session-based assistant | Ongoing coaching, project copilots, iterative work | Chat sessions with session memory |

---

## 1. Single Agent + Tools

The simplest and most common pattern. One agent handles the entire request, calling tools as needed.

### When to use

- Looking up information in your vault
- Drafting or editing notes
- Running simple tasks (create a note, search for content, query tasks)
- Any straightforward question-and-answer interaction

### How it works

Open the chat view and type your request. The active agent has access to 40+ built-in tools covering vault operations, task management, web search, and more. The agent decides which tools to call, executes them, and returns a response.

### Example interactions

```
You: "Summarize my meeting notes from this week"
Agent: [calls search_notes, batch_read_notes, send_to_chat]

You: "Create a project kickoff note for Project Alpha"
Agent: [calls create_note with structured template]

You: "What tasks are overdue in my Projects folder?"
Agent: [calls list_tasks with filter, formats results]
```

### Customizing the agent

Create a `.agent.md` file in your vault to define a specialist:

```yaml
---
name: Project Tracker
description: Tracks project status and generates updates
tools:
  - search_notes
  - list_tasks
  - create_note
  - get_recent_changes
---

You are a project tracking assistant. When asked about project status:
1. Search for the project folder
2. List recent changes and open tasks
3. Summarize progress and blockers
```

Place this file in any folder your vault's agent directories include (typically `Reference/Agents/`). It appears in the agent selector dropdown in the chat toolbar.

---

## 2. Sequential Pipeline

Chain multiple agents in a fixed order where each agent's output feeds into the next.

### When to use

- Research → Analysis → Draft → Review workflows
- Weekly or monthly report generation
- Multi-step document processing
- Any workflow where stage order matters and each stage builds on the prior one

### How to invoke

**As an automation action** — runs unattended on a trigger:

```yaml
---
name: Weekly report pipeline
triggers:
  - type: schedule
    schedule: "0 17 * * 5"
actions:
  - type: run-pipeline
    stages:
      - name: Research
        agentName: researcher
        promptTemplate: "Gather facts about vault activity this week"
      - name: Analysis
        agentName: analyst
        promptTemplate: "Analyze these findings: {{previousOutput}}"
      - name: Draft
        agentName: writer
        promptTemplate: "Write a concise weekly report from: {{previousOutput}}"
      - name: Review
        agentName: reviewer
        promptTemplate: "Review and improve this report: {{previousOutput}}"
    synthesisAgent: editor
---
```

**As a tool call** — an agent invokes it mid-conversation:

```
You: "Run a research pipeline on Q1 performance"
Agent: [calls run_pipeline with stages for researcher → analyst → writer]
```

### Prompt template placeholders

| Placeholder | Description |
|-------------|-------------|
| `{{previousOutput}}` | Output of the immediately preceding stage |
| `{{stage[N].output}}` | Output of stage at index N |
| `{{input.key}}` | Value from the pipeline input object |

### Retry and backoff

Each stage supports configurable retry with exponential backoff:

```yaml
stages:
  - name: Research
    agentName: researcher
    promptTemplate: "Research {{input.topic}}"
    maxRetries: 3
    timeoutMs: 180000
    retryPolicy:
      maxRetries: 3
      initialDelayMs: 2000
      backoffMultiplier: 2
      maxDelayMs: 30000
```

Without a `retryPolicy`, retries are immediate. With one, delays double between attempts (with ±10% jitter to prevent thundering herds).

---

## 3. Concurrent / Parallel Execution

Run multiple agents simultaneously and merge results. Reduces latency when gathering information from independent sources.

### When to use

- Broad information gathering from multiple sources
- Multi-source investigation (repo + calendar + notes + CRM)
- Any workflow where sub-tasks are independent and can run simultaneously
- Building prep briefs that pull from different data streams

### How to invoke

**As an automation action:**

```yaml
actions:
  - type: parallel
    actions:
      - type: run-agent
        agentId: repo-scanner
        input:
          task: "Check recent commits and open PRs"
      - type: run-agent
        agentId: calendar-scanner
        input:
          task: "List upcoming meetings this week"
      - type: run-agent
        agentId: note-scanner
        input:
          task: "Find recently modified project notes"
```

**As a tool call:**

```
You: "Build me a customer prep brief for the Acme meeting"
Agent: [calls run_parallel_agents with tasks for doc-reader, github-checker, meeting-history-scanner]
```

### How concurrency works

Vault Copilot runs a bounded session pool (default: 4 concurrent sessions). When you launch parallel tasks:

1. Each task acquires a pooled session
2. All tasks execute via `Promise.all()`
3. Results are collected and returned as an array
4. If the pool is full, tasks queue until a slot opens
5. Sessions idle-timeout after 5 minutes

Each task supports retry with an optional fallback prompt:

```json
{
  "tasks": [
    {
      "agentName": "researcher",
      "prompt": "Find market trends",
      "maxRetries": 2,
      "fallbackPrompt": "Summarize what you know about market trends"
    }
  ]
}
```

---

## 4. Agent Handoff

One agent transfers control to a specialist. The conversation continues seamlessly — message history is preserved across handoffs.

### When to use

- Multi-domain assistants where different experts handle different topics
- Routing from a triage agent to a domain specialist
- When one agent finishes its part and another should take over
- Support desk patterns with specialist escalation

### How to configure

Define handoffs in your `.agent.md` frontmatter:

```yaml
---
name: Triage Agent
description: Routes requests to the right specialist
handoffs:
  - label: "Research this topic"
    agent: researcher
    prompt: "Research the following: {{result}}"
    send: false
    model: GPT-5.1
  - label: "Draft a document"
    agent: writer
    prompt: "Draft a document about: {{result}}"
    send: true
canHandoffTo:
  - researcher
  - writer
  - reviewer
---
```

### How it works in practice

1. You chat with the Triage Agent
2. After the agent responds, **handoff buttons** appear below the message
3. Click a button to switch to the target agent
4. The target agent receives the conversation context + handoff prompt
5. If `send: true`, the handoff prompt sends automatically
6. The agent can also call `handoff_to_agent` programmatically

### Safety rails

The `canHandoffTo` field restricts which agents are valid handoff targets. If an agent tries to hand off to an agent not in its allow list, the handoff is blocked.

---

## 5. Manager / Planner

A manager agent dynamically creates a plan, delegates tasks to specialists, evaluates results, and replans if needed. The most powerful pattern for open-ended, complex tasks.

### When to use

- Open-ended goals where you don't know the exact steps upfront
- Complex work requiring research + coding + validation
- Tasks where specialist feedback should influence the next steps
- "Figure out why X failed, fix it, and verify the fix"

### How to invoke

An agent calls the `create_plan` tool:

```
You: "Figure out why the deployment failed, propose a fix, and open a PR draft"
Agent: [calls create_plan with goal]
```

The manager then:

1. **Plans** — Asks a planner agent to decompose the goal into tasks with dependencies
2. **Executes** — Runs ready tasks in parallel via the session pool
3. **Evaluates** — Asks the planner if results adequately address the goal
4. **Adjusts** — Redoes tasks or adds new ones based on evaluation feedback
5. **Replans** — If tasks fail, asks the planner for a revised approach (up to 3 times)
6. **Synthesizes** — Merges all task outputs into a final cohesive result

### Configuration

```json
{
  "goal": "Investigate deployment failure and propose fix",
  "plannerAgent": "planner",
  "synthesisAgent": "writer",
  "maxReplans": 3,
  "maxEvaluationRounds": 2,
  "taskTimeoutMs": 120000
}
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `maxReplans` | 3 | Maximum times the planner can revise the plan |
| `maxEvaluationRounds` | 2 | Maximum quality evaluation rounds after completion |
| `taskTimeoutMs` | 120000 | Per-task timeout in milliseconds |

### Quality evaluation loop

After all tasks complete successfully, the planner evaluates whether results actually address the original goal — not just whether they ran without errors. If the planner deems results insufficient, specific tasks can be marked for redo or new tasks added. This prevents the common failure mode where agents technically succeed but produce low-quality output.

### Agent capability matching

When creating a plan, the planner receives descriptions of available specialist agents (names, descriptions, and tools). This means the planner makes informed decisions about which agent should handle each task, rather than guessing agent names.

---

## 6. Group Discussion

Multiple agents collaborate in a shared thread, discussing a topic round-by-round. Unlike handoff (one-to-one), group discussion is many-to-many.

### When to use

- Brainstorming sessions requiring multiple perspectives
- Design reviews with architect, security, and product agents
- Tradeoff analysis (cost vs. security vs. usability)
- Problem decomposition where different viewpoints matter
- "Should we build this as local-first or cloud SaaS?"

### How to invoke

An agent calls the `group_discussion` tool:

```json
{
  "agents": ["architect", "security-analyst", "product-manager", "finance-analyst"],
  "topic": "Should we build this as a local-first Obsidian app or cloud SaaS?",
  "maxRounds": 3,
  "strategy": "round-robin",
  "synthesisAgent": "summarizer"
}
```

### Discussion strategies

| Strategy | How it works |
|----------|-------------|
| `round-robin` | Each agent speaks in order every round |
| `moderator-directed` | The first agent (moderator) decides who speaks next based on the discussion |

### Convergence detection

The discussion ends early if agents reach agreement (>60% signal consensus in their responses). This prevents unnecessary rounds when the group has already aligned.

### Synthesis

An optional synthesis agent summarizes the full thread — agreements, disagreements, and recommendations — into a structured output.

---

## 7. Human-in-the-Loop

The agent pauses for your approval before executing sensitive actions. Three layers of control: tool-level approval, workflow-level approval gates, and interactive questions.

### When to use

- Before modifying vault content (delete, rename, bulk updates)
- Before sending external communications (email, Telegram)
- Before executing deployment or infrastructure changes
- Any irreversible or high-risk action
- Workflows that need a human checkpoint before proceeding

### Tool approval configuration

**Session-level modes** (set via toolbar dropdown in the chat):

| Mode | Behavior |
|------|----------|
| Default | Respect per-tool approval settings |
| Bypass | Auto-approve all tools (development mode) |
| Autopilot | Auto-approve and iterate autonomously |

**Per-tool approval** (Settings → Tool Approval):

Set each tool to `auto`, `ask`, or `deny`. For example:
- `read_note` → **auto** (always safe)
- `delete_note` → **ask** (destructive)
- `send_telegram_message` → **ask** (external communication)

**Per-category defaults** (Settings → Tool Category Approval):

Apply a default to entire categories like READ, WRITE, AGENTIC, WEB. Per-tool settings override category defaults.

### Approval gates in automations

Insert an `approval-gate` action to pause a workflow for human decision:

```yaml
actions:
  - type: run-agent
    agentId: evidence-collector
  - type: approval-gate
    message: "Evidence pack is ready for review. Approve to finalize?"
  - type: run-prompt
    promptId: generate-audit-report
  - type: notify
    channel: telegram
    message: "Audit report finalized"
```

The workflow checkpoints its state before pausing. If you close Obsidian and reopen later, the approval prompt reappears and the workflow resumes from where it left off.

### Interactive questions

Agents can ask structured questions mid-conversation using the `ask_question` tool:

```
Agent: [calls ask_question with type="radio", question="Which section to update?", options=["Morning", "Evening", "Tasks"]]
```

This renders a modal with radio buttons, checkboxes, or free-text input.

---

## 8. Guardrails

Validation layers that check input/output before expensive or risky actions happen. Guardrails run automatically — no user action needed once configured.

### When to use

- Preventing PII from leaking in agent responses
- Enforcing brand/style guidelines on generated content
- Blocking harmful or off-topic requests
- Transforming content to meet policy requirements
- Rate-limiting tool calls or enforcing token budgets

### How to configure

Create a `.guardrail.md` file in your vault:

```yaml
---
name: PII Blocker
description: Block responses containing personal identifiable information
type: output
action: block
enabled: true
patterns:
  - "\\b\\d{3}-\\d{2}-\\d{4}\\b"
  - "\\b[A-Z0-9._%+-]+@[A-Z0-9.-]+\\.[A-Z]{2,}\\b"
---

Check the agent's response for any personal identifiable information
including social security numbers, email addresses, phone numbers,
or physical addresses. If found, block the response.
```

### Guardrail types

| Type | When it runs | Use case |
|------|-------------|----------|
| Input | Before the user's message reaches the agent | Block harmful prompts, validate format |
| Output | After the agent responds, before showing to user | PII removal, style enforcement, safety |

### Guardrail strategies

| Strategy | How it works | Speed |
|----------|-------------|-------|
| Regex-based | Pattern matching against `patterns` list | Fast |
| LLM-based | AI evaluation using the markdown body instructions | Slower but more nuanced |
| Custom expression | Programmable verdict logic | Configurable |

### Verdicts

| Verdict | Effect |
|---------|--------|
| `pass` | Content proceeds normally |
| `block` | Content is rejected with an explanation |
| `transform` | Content is modified before proceeding |

### Operational guardrails

Beyond content guardrails, Vault Copilot includes:

- **Rate limiter** — Prevents runaway tool calls within a time window
- **Token budget tracker** — Caps total tokens per session or workflow

---

## 9. Durable / Long-Running Workflows

Workflows that can pause, checkpoint, survive plugin reloads and app restarts, and resume later.

### When to use

- Workflows that wait for external events (file creation, HTTP endpoint readiness)
- Multi-day investigations with human approval checkpoints
- Processes that depend on DNS propagation, CI builds, or external systems
- Onboarding/offboarding sequences with multiple wait steps
- Any workflow where you might close Obsidian and come back later

### Wait actions

**Wait for external condition:**

```yaml
- type: wait-for
  conditionType: file-exists
  target: "Projects/alpha/approval.md"
  timeoutMs: 86400000  # 24 hours
```

| Condition type | How it detects | Mechanism |
|---------------|----------------|-----------|
| `file-exists` | Vault event listener + 60s safety poll | Event-driven (instant) |
| `expression` | Metadata cache event listener + configurable poll | Event-driven |
| `http` | Adaptive polling (doubles interval, caps at 5 min) | Polling |

**Wait for fixed duration:**

```yaml
- type: wait-duration
  durationMs: 1800000
  message: "Waiting 30 minutes for DNS propagation..."
```

### Checkpoint and resume

Every wait action and approval gate **checkpoints workflow state to disk** before pausing. The checkpoint includes:

- Current step index
- Results from all completed steps
- The trigger that started the workflow
- A composite key (`automationId:executionId`) for multi-workflow support

**On plugin reload or app restart:**

1. Vault Copilot detects pending checkpoints (status: `paused` or `waiting`)
2. After a 5-second stabilization delay, it auto-resumes each workflow
3. Completed steps are skipped (idempotent replay)
4. The workflow continues from where it left off

**Checkpoint management:**

- Checkpoints older than 24 hours trigger a warning log
- Checkpoints older than 7 days are automatically cleaned up
- You can manually resume a checkpoint via `engine.resumeFromCheckpoint(key)`

### Retry with exponential backoff

Any automation action can be configured with a retry policy:

```yaml
- type: run-agent
  agentId: data-fetcher
  retryPolicy:
    maxRetries: 3
    initialDelayMs: 1000
    backoffMultiplier: 2
    maxDelayMs: 30000
    retryableErrors:
      - "timeout"
      - "ECONNREFUSED"
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `maxRetries` | 1 | Maximum retry attempts |
| `initialDelayMs` | 1000 | Delay before first retry (ms) |
| `backoffMultiplier` | 2 | Multiply delay each retry |
| `maxDelayMs` | 30000 | Maximum delay cap (ms) |
| `retryableErrors` | *(all)* | Error substrings to retry on |

Delay formula: `min(initialDelayMs × multiplier^attempt, maxDelayMs)` with ±10% random jitter.

---

## 10. Session-Based Assistant

The agent maintains working memory and context across a conversation thread. Sessions survive across multiple interactions.

### When to use

- Project copilots that need to remember context across messages
- Iterative workflows where you refine results over multiple turns
- Voice agents maintaining conversation state
- Daily operations agents that remember your open tasks and decisions

### Session memory

Agents can explicitly store and recall facts during a conversation:

```
You: "Remember that the deployment deadline is March 15"
Agent: [calls save_memory with key="deployment_deadline", value="March 15"]

... later in the conversation ...

You: "When is the deployment deadline?"
Agent: [calls recall_memory] → "March 15"
```

Session memory is structured:

| Field | Purpose |
|-------|---------|
| `facts` | Key-value pairs of learned information |
| `metadata` | Arbitrary state attached to the session |
| `openItems` | Todo items tracked per-session |
| `planSummary` | Current plan or strategy |

### Session lifecycle

1. **Create** — New session starts with agent, tools, system prompt, MCP servers
2. **Interact** — Message history accumulates across turns
3. **Resume** — Sessions can be restored with full history + agents
4. **Destroy** — Sessions clean up on idle timeout (5 min) or shutdown

### Managing sessions

- **Session panel** (left sidebar) shows all saved conversations
- **New chat** creates a fresh session
- **Switch agents** mid-session via the toolbar dropdown
- **Archive** sessions from the right-click menu
- **Context summarization** compresses long conversations to stay within token limits

---

## Automation Engine

The automation engine is the runtime that executes all workflow patterns. It watches for triggers, runs actions, and manages lifecycle.

### Defining automations

Create `.automation.md` files anywhere in your vault:

```yaml
---
name: My Automation
description: What this automation does
enabled: true
triggers:
  - type: schedule
    schedule: "0 9 * * *"
actions:
  - type: run-agent
    agentId: daily-planner
    input:
      task: "Create today's plan"
---

# My Automation

Detailed instructions for what the agent should do.
This markdown body is passed as context to the agent.
```

### Trigger types

| Trigger | Fires when | Example |
|---------|-----------|---------|
| `schedule` | Cron expression matches | `"0 9 * * 1-5"` (weekdays 9am) |
| `file-created` | File matching pattern is created | `"Projects/**/*.md"` |
| `file-modified` | File matching pattern is modified | `"Daily Notes/*.md"` |
| `file-deleted` | File matching pattern is deleted | `"Archive/**/*"` |
| `tag-added` | Tag is added to any note | `"#review"` |
| `vault-opened` | Vault is opened | — |
| `startup` | Plugin loads | — |

### Action types

| Action | Purpose | Supports retry |
|--------|---------|---------------|
| `run-agent` | Execute an AI agent | Yes |
| `run-prompt` | Execute a prompt template | Yes |
| `run-skill` | Execute a skill | Yes |
| `approval-gate` | Pause for human approval | No (self-managed) |
| `conditional` | Branch on condition | No |
| `parallel` | Run actions concurrently | No (self-managed) |
| `wait-for` | Wait for external condition | No (self-managed) |
| `wait-duration` | Pause for fixed time | No (self-managed) |
| `run-pipeline` | Sequential agent pipeline | No (stages have own retry) |
| `notify` | Send Telegram notification | Yes |

### Inter-step data flow

Actions can reference results from previous steps using `{{step[N].result}}` placeholders:

```yaml
actions:
  - type: run-agent
    agentId: researcher
    input:
      task: "Find relevant data"
  - type: run-agent
    agentId: writer
    input:
      task: "Write a summary based on: {{step[0].result}}"
  - type: notify
    channel: telegram
    message: "Summary ready:\n{{step[1].result}}"
```

### Monitoring automations

- **Settings → Automations** shows all registered automations
- Each automation displays: name, schedule, last run, next run, execution count
- **Workflow Status** section shows active/paused workflows with checkpoint details
- Execution history is logged (last 100 entries)

---

## Configuration Reference

### File types

| Extension | Purpose | Where to place |
|-----------|---------|---------------|
| `.agent.md` | Custom agent definitions | Agent directories (e.g., `Reference/Agents/`) |
| `.automation.md` | Automation workflows | Anywhere in vault |
| `.prompt.md` | Reusable prompt templates | Prompt directories |
| `SKILL.md` | Skill definitions | `skills/<name>/SKILL.md` |
| `.guardrail.md` | Input/output guardrails | Anywhere in vault |
| `.instructions.md` | Session instructions | Instruction directories |

### Agent frontmatter fields

```yaml
---
name: Agent Name                  # Required
description: What this agent does # Required
model: gpt-4o                     # Optional: model override
tools: [tool1, tool2]             # Optional: tool allowlist
skills: [skill-name]              # Optional: skills to load
handoffDescription: "..."         # Optional: shown to other agents
handoffs:                         # Optional: outbound handoffs
  - label: "Button Label"
    agent: target-name
    prompt: "Context to send"
    send: false
    model: "GPT-5.1"
canHandoffTo: [agent1, agent2]    # Optional: safety rail
userInvokable: true               # Optional: show in menus
argumentHint: "Hint text"         # Optional: input hint
---
```

### Automation frontmatter fields

```yaml
---
name: Automation Name             # Required
description: What it does         # Optional
enabled: true                     # Default: true
run-on-install: false             # Default: false
triggers: [...]                   # Required: at least one trigger
actions: [...]                    # Required: at least one action
---
```

### RetryPolicy fields

```yaml
retryPolicy:
  maxRetries: 3                   # Default: 1
  initialDelayMs: 1000            # Default: 1000
  backoffMultiplier: 2            # Default: 2
  maxDelayMs: 30000               # Default: 30000
  retryableErrors:                # Optional: if unset, all errors retry
    - "timeout"
    - "ECONNREFUSED"
```

### MCP server fields (stdio)

```json
{
  "id": "server-id",
  "name": "Display Name",
  "transport": "stdio",
  "command": "node",
  "args": ["path/to/server.js"],
  "env": {"KEY": "value"},
  "enabled": true,
  "companionProcess": {
    "command": "node",
    "args": ["auth-server.js"],
    "startDelayMs": 1500
  }
}
```

### MCP server fields (HTTP)

```json
{
  "id": "server-id",
  "name": "Display Name",
  "transport": "http",
  "url": "https://mcp.example.com",
  "apiKey": "secret",
  "headers": {"Authorization": "Bearer token"},
  "enabled": true
}
```

---

## Practical Workflow Recipes

### Executive operations agent

**Pattern:** Router + parallel fetch + sequential pipeline + approval

```yaml
---
name: Daily executive brief
triggers:
  - type: schedule
    schedule: "0 7 * * 1-5"
actions:
  - type: parallel
    actions:
      - type: run-agent
        agentId: calendar-scanner
        input: { task: "Summarize today's meetings" }
      - type: run-agent
        agentId: email-scanner
        input: { task: "Summarize inbox highlights" }
      - type: run-agent
        agentId: task-scanner
        input: { task: "List overdue and due-today tasks" }
  - type: run-pipeline
    stages:
      - name: Merge
        agentName: writer
        promptTemplate: "Create a daily brief from: {{previousOutput}}"
      - name: Polish
        agentName: editor
        promptTemplate: "Edit for clarity: {{previousOutput}}"
  - type: approval-gate
    message: "Brief ready. Send to Telegram?"
  - type: notify
    channel: telegram
    message: "{{step[1].result}}"
---
```

### Research-to-note pipeline

**Pattern:** Sequential pipeline with synthesis

```yaml
---
name: Research pipeline
triggers:
  - type: tag-added
    tag: "#research"
actions:
  - type: run-pipeline
    stages:
      - name: Collect
        agentName: researcher
        promptTemplate: "Research the topic in this note thoroughly"
      - name: Analyze
        agentName: analyst
        promptTemplate: "Extract key themes from: {{previousOutput}}"
      - name: Draft
        agentName: writer
        promptTemplate: "Write a structured research note from: {{previousOutput}}"
    synthesisAgent: reviewer
---
```

### Project copilot with handoffs

**Pattern:** Router agent with specialist handoffs

```yaml
---
name: Project Assistant
description: Routes project questions to specialists
tools: [search_notes, list_tasks, get_recent_changes]
handoffs:
  - label: "Investigate in GitHub"
    agent: github-investigator
    prompt: "Check the repo for: {{result}}"
  - label: "Draft an update"
    agent: project-writer
    prompt: "Draft a project update based on: {{result}}"
  - label: "Plan next sprint"
    agent: sprint-planner
    prompt: "Plan the next sprint considering: {{result}}"
canHandoffTo: [github-investigator, project-writer, sprint-planner]
---

You are the project assistant. Analyze project status and route to specialists.
```

### Multi-day workflow with external waits

**Pattern:** Durable workflow with wait-for and approval

```yaml
---
name: Onboarding workflow
triggers:
  - type: tag-added
    tag: "#new-hire"
actions:
  - type: run-agent
    agentId: hr-assistant
    input: { task: "Create onboarding checklist" }
  - type: wait-for
    conditionType: file-exists
    target: "HR/approvals/{{step[0].result}}"
    timeoutMs: 172800000
  - type: run-agent
    agentId: it-provisioner
    input: { task: "Provision accounts" }
    retryPolicy:
      maxRetries: 3
      initialDelayMs: 5000
      backoffMultiplier: 2
      maxDelayMs: 60000
  - type: wait-for
    conditionType: http
    target: "https://provisioning.internal/status"
    expectedValue: "complete"
    pollIntervalMs: 60000
    timeoutMs: 3600000
  - type: approval-gate
    message: "All accounts provisioned. Approve to send welcome email?"
  - type: notify
    channel: telegram
    message: "Onboarding complete for new hire"
---
```

---

## Choosing the Right Pattern

```
Is the task straightforward?
  → Yes → Single agent + tools

Does it have a fixed sequence of steps?
  → Yes → Sequential pipeline

Can sub-tasks run independently?
  → Yes → Parallel execution (+ optional synthesis)

Do different domains need different expertise?
  → Yes → Agent handoff or router + specialists

Is the goal open-ended with uncertain steps?
  → Yes → Manager / planner

Do you need multiple perspectives on a decision?
  → Yes → Group discussion

Does the workflow involve risky/external actions?
  → Yes → Add human-in-the-loop (approval gates, tool approval)

Might the workflow span hours or days?
  → Yes → Add durable wait actions (wait-for, wait-duration)

Need to enforce policies on input/output?
  → Yes → Add guardrails

Is it an ongoing conversation across sessions?
  → Yes → Session-based assistant with memory
```

Most real-world workflows combine 2-3 patterns. The sweet spot for most practical systems is:

**Router + 2-4 specialists + human approval for risky steps**

This is easier to debug than open-ended group chat, while still providing specialization and tool use.
