# AI Funnel Anomaly Analyzer — Agentic System (Stage 1)

A production-style agentic AI system that autonomously investigates product funnel drops,
identifies root causes, and traces its own reasoning — built to understand the fundamentals
of agentic AI architecture from the ground up.

---

## What This Project Is

Most AI demos chain two LLM calls together and call it an "agent." This project is different.

This is a genuine agentic system where the LLM autonomously decides:
- Which tools to call
- In what order
- With what arguments
- When it has enough information to stop

The LLM drives the investigation. Python just executes its decisions.

---

## The Problem Being Solved

A product funnel has four stages: Visit → Signup → Activation → Purchase.

When purchase conversion drops, a human analyst would manually slice the data across
multiple dimensions — time, device, traffic source — to find the root cause.
This system does that investigation autonomously, in under 10 seconds, with a full
reasoning trace showing every step it took.

---

## Architecture Overview

```
User Goal (plain English)
        ↓
   System Prompt
   (defines agent behaviour)
        ↓
┌─────────────────────────────────────┐
│         THE REACT LOOP              │
│                                     │
│  LLM receives full message history  │
│           ↓                         │
│   finish_reason == "tool_calls"?    │
│     YES ↓              NO ↓         │
│  Dispatcher        Final Answer     │
│  runs tool         Loop Ends        │
│     ↓                               │
│  Result added                       │
│  to history                         │
│     ↓                               │
│  Loop repeats                       │
└─────────────────────────────────────┘
```

The key insight: the LLM sees the full conversation history on every iteration.
This is what enables cross-step reasoning — using Tool 2's output as input to Tool 3.

---

## The 6 Layers of the System

### Layer 1 — Dataset (Cell 3)
Synthetic funnel data for 10,000 users across January 2025.
Each user has: device, traffic source, experiment group, date, and stage reached.

A hidden anomaly is injected: mobile users who reached purchase after Jan 20
are pushed back to activation — simulating a broken mobile checkout flow.

**Funnel conversion rates set:**
- Visit → Signup: 70%
- Signup → Activation: 50%
- Activation → Purchase: 40% (expected), ~30% (actual, due to anomaly)

---

### Layer 2 — Tool Functions (Cell 4)
Four Python functions that query the dataset. These are the "lab equipment."
The LLM cannot run these directly — it requests them, and the dispatcher executes them.

| Tool | Purpose |
|---|---|
| `get_funnel_summary()` | Overall conversion rates at each stage |
| `detect_anomalies()` | Which dates had abnormal conversion drops |
| `get_device_breakdown(start_date, end_date)` | Mobile vs desktop conversion rates |
| `get_traffic_source_breakdown(start_date, end_date)` | Ads vs organic conversion rates |

**Key design principle:** Tools are dumb and reliable. Intelligence lives in the LLM.
Tools accept flexible parameters (date ranges) rather than hardcoded values,
so the agent can discover values dynamically and pass them in.

---

### Layer 3 — Tool Registry / Schemas (Cell 5)
JSON descriptions of every tool sent to OpenAI with every API call.
This is how the LLM knows what tools exist and when to use them.

```python
{
    "type": "function",
    "function": {
        "name": "get_device_breakdown",
        "description": "Returns conversion rates split by device type: mobile vs desktop.
                        Use start_date from anomaly detection results to filter
                        to the affected period.",
        "parameters": {
            "start_date": {"type": "string", "description": "YYYY-MM-DD format"}
        }
    }
}
```

**Critical insight:** The description quality directly controls agent behaviour.
Vague descriptions → wrong tool selection. Clear descriptions → correct sequencing.
Writing tool descriptions is a product management skill, not just an engineering one.

---

### Layer 4 — Tool Dispatcher (Cell 6)
Pure Python if/else router. Zero intelligence. Zero LLM.

Receives the LLM's tool request → executes the Python function → returns the result.

```
LLM says: "call get_device_breakdown with start_date 2025-01-21"
          ↓
Dispatcher receives: {"name": "get_device_breakdown", "arguments": {"start_date": "2025-01-21"}}
          ↓
Dispatcher runs: get_device_breakdown(start_date="2025-01-21")
          ↓
Dispatcher returns: {"mobile": {"rate": 0.0}, "desktop": {"rate": 0.398}}
```

Includes a safety net for hallucinated tool names — returns a clean error
with a list of available tools instead of crashing.

---

### Layer 5 — The ReAct Agent Loop (Cell 7)
The brain of the system. This is what makes it genuinely agentic.

**How it works:**
1. Send user goal + system prompt + full conversation history to LLM
2. LLM responds with either:
   - `finish_reason: "tool_calls"` → execute tool, add result to history, repeat
   - `finish_reason: "stop"` → return final answer, end loop
3. Repeat until stop or max_iterations reached

**The conversation history grows every iteration:**
```
Iteration 1: [system prompt, user goal]
Iteration 2: [system prompt, user goal, Tool1 request, Tool1 result]
Iteration 3: [system prompt, user goal, Tool1 request, Tool1 result, Tool2 request, Tool2 result]
```

This growing history is what enables cross-step reasoning.
The LLM used anomaly dates from Tool 2 as date filters in Tool 3 — without being told to.

---

### Layer 6 — Agent Execution (Cell 8)
Single function call. The agent handles everything else.

```python
result = run_agent(
    "Our purchase conversion has dropped significantly. "
    "Investigate the funnel, identify when the drop started, "
    "which user segments are affected, "
    "and what the most likely root cause is."
)
```

---

## What the Agent Actually Did (Real Output)

The agent completed the investigation in 4 iterations, 9 reasoning steps:

```
ITERATION 1 → called get_funnel_summary
              Saw activation→purchase rate at 30.4% vs expected 40%
              Confirmed: something is wrong

ITERATION 2 → called detect_anomalies
              Found: 10 anomaly days, all after Jan 21
              Confirmed: problem started Jan 21

ITERATION 3 → called get_device_breakdown (start_date: 2025-01-21)
              Mobile: 0 purchases from 839 activations = 0% rate
              Desktop: 154 purchases from 387 activations = 39.8% rate

           → called get_traffic_source_breakdown (start_date: 2025-01-21)
              Ads: 14% | Organic: 11.6% — both equally affected
              Confirmed: not a marketing problem

ITERATION 4 → finish_reason: stop
              Final diagnosis delivered
```

**The agent correctly identified:**
- What: Purchase conversion collapsed
- When: Starting January 21st
- Who: Mobile users only — desktop completely healthy
- Why not: Traffic source — both channels equally affected
- Root cause: Mobile-specific technical issue (the injected anomaly)

---

## Key Concepts Learned

### What Makes This Genuinely Agentic

| Your original app.py | This system |
|---|---|
| You decided the sequence in Python | LLM decides the sequence |
| Fixed 2-step chain | Variable steps — agent decides when to stop |
| LLM received text, returned text | LLM requests tools, observes results |
| No reasoning trace | Full trace of every decision |
| Could not handle novel questions | Handles any question within tool scope |

---

### The 3 Layers of Failure (Debugging Mental Model)

When an agentic system gives wrong output, the problem lives in one of three places:

**Intelligence Layer** — LLM + system prompt
- Hallucination in reasoning
- Vague or missing system prompt instructions
- Wrong model choice for the task

**Execution Layer** — Tool schemas + dispatcher + tool functions
- Tool description too vague → LLM picks wrong tool
- Dispatcher routing error → wrong function called
- Tool function bug → correct request, wrong data returned

**Data Layer** — Underlying dataset
- Stale data → correct analysis of outdated information
- Incomplete data → agent misses patterns that exist
- Sampling bias → agent draws wrong population-level conclusions

**Debugging order:**
1. Read the reasoning trace — did agent call right tools in right order?
2. Run tools manually — did they return correct data?
3. Check the data — is it fresh and complete?
4. Check tool descriptions — did agent skip a tool it should have used?
5. Check system prompt — did agent follow the right sequence?
6. Check the model — did LLM misinterpret correct tool output?

---

### Why Tool-Based Agents Have a Ceiling

The agent can only investigate dimensions you've pre-built tools for.
If a new type of anomaly occurs that no existing tool covers, the agent will
give a shallow or incorrect answer.

**This is a core PM insight:** The tool library defines the agent's capability boundary.
Growing the tool library based on recurring incident patterns is an ongoing product decision.

**The three categories of problems:**
- Known knowns — have happened before, tool exists, agent handles well
- Known unknowns — could happen, tool not built yet, PM decides if worth building preemptively
- Unknown unknowns — never imagined, no tool, agent fails silently

---

### Single LLM vs Multi-Agent Orchestration

This system uses a single LLM acting as its own agent.
The same model both decides which tools to call AND synthesises the final answer.

In a multi-agent system:
```
Orchestrator LLM
├── Specialist Agent 1 (diagnosis) — own system prompt, own tools
├── Specialist Agent 2 (root cause) — own system prompt, own tools
└── Specialist Agent 3 (recommendations) — own system prompt, own tools
```

Each specialist is a separate LLM call with narrower scope.
Multi-agent systems handle greater complexity but introduce:
- Higher cost (multiple LLM calls per query)
- Higher latency (sequential or parallel agent hops)
- Error compounding risk (bad output from Agent 1 corrupts Agent 2)

This Stage 1 system is the foundation. Every agent in a multi-agent
system works exactly like this — just with a narrower scope.

---

## What's Missing (Stage 2, 3, 4)

This is a Stage 1 system. Real production systems need:

| Limitation | What it means | Fixed in |
|---|---|---|
| No memory | Agent starts from zero every run | Stage 3 |
| No human approval gate | Agent acts without confirmation | Stage 2 |
| No confidence scores | All conclusions stated with equal certainty | Stage 4 |
| No failure case handling | Novel incidents without tools get shallow answers | Stage 4 |
| No persistent logging | Reasoning traces lost after session | Stage 2 |

---

## Tools and Stack

| Tool | Purpose |
|---|---|
| Python 3 | Core language |
| OpenAI API (gpt-4o-mini) | LLM for reasoning and tool selection |
| Pandas | Dataset creation and querying |
| NumPy | Random data generation |
| Google Colab | Development environment |
| JSON | Tool schema definitions and data serialisation |

---

## How to Run

1. Open Google Colab
2. Run Cell 1: `!pip install openai pandas numpy`
3. Run Cell 2: Set up OpenAI API key using Colab secrets
4. Run Cells 3–7: Build dataset, tools, registry, dispatcher, agent loop
5. Run Cell 8: Fire the agent with a plain English investigation goal
6. Watch the reasoning trace print in real time

---

## Interview Talking Points

If asked to explain this project in 2 minutes:

"I built a genuine agentic AI system — not a two-step LLM chain, but a system
where the LLM autonomously decides which tools to call, in what order, with what
arguments, and when it has enough information to stop.

The system investigates product funnel drops by giving an LLM access to four
analytical tools — funnel summary, anomaly detection, device breakdown, and
traffic source breakdown. When given a plain English goal, the agent calls tools
sequentially, uses observations from earlier steps as inputs to later steps,
and produces a complete diagnosis with a full reasoning trace.

The most interesting moment is iteration 3 — the agent used anomaly dates
discovered in Tool 2 as date filters when calling Tool 3. That cross-step
reasoning happened autonomously — I didn't code that sequence. The LLM decided it.

The key PM insight I took from building this: the agent is only as capable as
its tools, and only as intelligent as its system prompt. Both are product
decisions, not engineering decisions."

---

*Stage 1 complete. Stage 2 adds human approval gates and persistent reasoning logs.*
