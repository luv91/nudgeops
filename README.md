# NudgeOps

[![PyPI version](https://badge.fury.io/py/nudgeops.svg)](https://pypi.org/project/nudgeops/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Tests](https://github.com/luv91/nudgeops/actions/workflows/test.yml/badge.svg)](https://github.com/luv91/nudgeops/actions/workflows/test.yml)

**Runtime semantic guardrails for AI agents.**

Detect loops. Nudge agents back on track. Stop runaway costs.

## Why NudgeOps?

AI agents get stuck in loops. Each loop iteration costs money (API calls, tokens). NudgeOps detects and stops loops before they waste your budget.

```
Without NudgeOps:
search("XYZ-9999") → not found
search("XYZ-9999") → not found  ($0.02)
search("XYZ-9999") → not found  ($0.04)
... 50 more attempts ...        ($1.00+ wasted)

With NudgeOps:
search("XYZ-9999") → not found
search("XYZ-9999") → BLOCKED: "Same action failed twice. Try a different approach."
```

## Installation

```bash
pip install nudgeops

# With LangGraph support
pip install nudgeops[langgraph]

# With OpenAI for thought normalization
pip install nudgeops[openai]

# Everything
pip install nudgeops[all]
```

## Quick Start

### Basic Usage (Pattern Detection)

```python
from nudgeops import UniversalGuard

guard = UniversalGuard()

# In your agent loop
result = guard.check(state)
if result.blocked:
    print(f"Loop detected! {result.reason}")
```

### Smart Guard (Intent-Level)

```python
from nudgeops import SmartNudgeOps, MockLLMClient

nudgeops = SmartNudgeOps(llm_client=MockLLMClient())

result = nudgeops.check(
    state={"page": "search"},
    thought="I should search for XYZ-9999",
    tool_name="search",
    args={"query": "XYZ-9999"}
)

if result.blocked:
    print(f"Blocked: {result.reason}")
    print(f"Nudge: {result.nudge_message}")
```

### LangGraph Integration

```python
from nudgeops import SmartNudgeOps
from langgraph.graph import StateGraph

builder = StateGraph(AgentState)
builder.add_node("agent", agent_node)
builder.add_node("tools", tool_node)

# One line to add protection
nudgeops = SmartNudgeOps(llm_client=my_llm)
nudgeops.apply(builder)

graph = builder.compile()
```

## Protection Mechanisms (15 Total)

NudgeOps uses a **multi-layered, redundant detection system**. Even if one detector misses something, others catch it.

### Core Layer (v0.1) - Pattern-Based Detection

| # | Mechanism | What It Detects | Threshold |
|---|-----------|-----------------|-----------|
| 1 | **Stutter Detector** | Exact same action repeated | 3+ identical actions |
| 2 | **Insanity Detector** | Semantically similar actions (embeddings) | ≥0.85 similarity, 3+ times |
| 3 | **Phantom Progress** | Different actions but state frozen | 2+ actions, same state |
| 4 | **Ping-Pong Detector** | Circular agent handoffs (A→B→A→B) | 2x pattern repetition |
| 5 | **Loop Scorer** | Aggregates scores with decay | 2.0→NUDGE, 3.0→STOP |
| 6 | **Intervention Manager** | Generates nudge messages | 4 templates per loop type |

### Smart Layer (v0.2) - Intent-Level Protection

| # | Mechanism | Purpose |
|---|-----------|---------|
| 7 | **Smart Guard L1** | Exact action repeat detection → WARN at 2, BLOCK at 3 |
| 8 | **Smart Guard L2** | Intent exhaustion → BLOCK after 3 strategy variations |
| 9 | **Thought Normalizer** | LLM converts diverse thoughts → 5-word canonical intents |
| 10 | **State Hasher** | Filters volatile keys (timestamp, uuid) for stable comparison |
| 11 | **Action Hasher** | Normalizes IDs, emails, UUIDs → patterns for matching |
| 12 | **Failure Memory** | Two-level tracking: action failures + intent clusters |
| 13 | **Failure Events** | 50+ error signatures for canonical failure classification |
| 14 | **Observability Layer** | Tracks blocks, tokens saved, cost saved in USD |
| 15 | **SmartNudgeOps** | LangGraph integration with `apply()` wrapper |

## How It Works

### Two-Level Blocking

**Level 1 (Action)**: Block exact action repeats after 2 attempts
```
search({q:"XYZ-9999"}) → fails
search({q:"XYZ-9999"}) → BLOCKED (same action twice)
```

**Level 2 (Intent)**: Block exhausted strategies after 3 variations
```
"search XYZ-9999"  → intent: "find product by ID"
"try XYZ9999"      → intent: "find product by ID" (same!)
"try XYZ 9999"     → intent: "find product by ID" (same!)
"try XYZ--9999"    → BLOCKED (intent exhausted, try different strategy)
```

## Observability & Metrics

Track your savings with built-in observability:

```python
from nudgeops import SmartNudgeOps, ObservabilityLayer

nudgeops = SmartNudgeOps(llm_client=my_llm)

# Run your agent...
# ...

# Check metrics
metrics = nudgeops.observability.get_metrics()
print(f"Loops blocked: {metrics['blocks']}")
print(f"Tokens saved: {metrics['tokens_saved']}")
print(f"Cost saved: ${metrics['cost_saved_usd']:.2f}")
```

## API Reference

### SmartNudgeOps

```python
from nudgeops import SmartNudgeOps

nudgeops = SmartNudgeOps(
    llm_client=my_llm,           # Required: LLM for thought normalization
    action_repeat_threshold=2,   # Block after N exact repeats (default: 2)
    intent_repeat_threshold=3,   # Block after N intent variations (default: 3)
)

# Check an action before executing
result = nudgeops.check(
    state=dict,          # Current agent state
    thought=str,         # Agent's reasoning/thought
    tool_name=str,       # Tool being called
    args=dict,           # Tool arguments
)

# result.decision: ALLOW, WARN, or BLOCK
# result.nudge_message: Explanation for the agent
# result.reason: Technical reason for blocking
```

### UniversalGuard (Core Pattern Detection)

```python
from nudgeops import UniversalGuard

guard = UniversalGuard()
result = guard.check(state)

# result.blocked: bool
# result.reason: str
# result.detections: list of DetectionResult
```

## License

MIT
