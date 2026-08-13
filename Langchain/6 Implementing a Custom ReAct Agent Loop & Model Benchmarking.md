# LangChain: Implementing a Custom ReAct Agent Loop & Model Benchmarking

## Metadata

- **Topic:** Custom ReAct Agent Loop Implementation, LangChain Tool Decorators, Model Benchmarking, and LangSmith Tracing
    
- **Difficulty:** Intermediate
    
- **Tags:** #langchain #ollama #pydantic #react-loop #agentic-ai #python #langsmith #model-benchmarking
    
- **Source:** Agentic AI Engineering with LangChain & LangGraph
    
- **Date:** 2026-08-05
    

## Executive Summary

- **Explicit ReAct Implementation:** Replaces high-level abstractions like `create_agent` with an explicit `while` loop that handles the reasoning cycle, tool execution, and context tracking.
    
- **LangChain Utility Functions:** Employs `init_chat_model` for string-based LLM instantiation and `@tool` decorators for converting Python functions into model-agnostic JSON Schema definitions.
    
- **Model Benchmarking Risks:** Demonstrates that swapping an LLM provider or upgrading to a newer model version (e.g., from `qwen3:1.7b` to `GPT-5.2`) can degrade agent performance without task-specific benchmarking.
    
- **Defensive System Prompting:** Uses strict system instructions to mitigate hallucinated values and force the agent to rely on actual tool execution.
    
- **LangSmith Instrumentation:** Wraps custom functions with the `@traceable` decorator to aggregate overall execution metrics, token usage, latency, and nested sub-calls.
    

## Main Concepts & Theory

### 1. Custom Agent Execution Architecture

Building an agent loop manually provides fine-grained control over context updating, iteration bounds, and fallback behaviors.

```mermaid
                        ┌────────────────────────────────────────┐
                        │      User Query + System Prompt        │
                        └───────────────────┬────────────────────┘
                                            │
                                            ▼
                       ┌──────────────────────────────────────────┐
  ┌───────────────────►│ 1. Thought Pass (init_chat_model)         │
  │                    │ Prompt model with current message trace  │
  │                    └────────────────────┬─────────────────────┘
  │                                         │
  │                                  Has tool_calls?
  │                                  /             \
  │                            YES  /               \  NO (Empty / Done)
  │                                /                 \
  │                               ▼                   ▼
  │            ┌────────────────────────────────┐  ┌────────────────────┐
  │            │ 2. Tool Lookup & Execution     │  │  Final Answer      │
  │            │ Dispatch tool by name via dict │  │  Return AI content │
  │            └────────────────┬───────────────┘  └────────────────────┘
  │                             │
  │                             ▼
  │            ┌────────────────────────────────┐
  │            │ 3. State & Trace Update        │
  │            │ Append AIMessage + ToolMessage │
  │            └────────────────┬───────────────┘
  │                             │
  └─────────────────────────────┘
```

### 2. The Model Swapping Myth & Benchmarking

While LangChain's unified interfaces allow changing models by updating a single string parameter (e.g., swapping `qwen3:1.7b` for `openai:gpt-5.2` via `init_chat_model`), doing so without evaluation can introduce failures.

#### Model Behavior Comparison

|**Dimension**|**Open-Weights (qwen3:1.7b)**|**SOTA Proprietary (GPT-5)**|**SOTA Variant (GPT-5.2)**|
|---|---|---|---|
|**Setup Cost**|Free local execution via Ollama.|Paid API call per token.|Paid API call per token.|
|**Tool Calling Stability**|Requires defensive prompting to avoid hallucinated parameters.|High native adherence to schemas and instructions.|Task-specific failure; asked clarifying questions instead of invoking tools.|
|**Latency & Cost**|Local hardware dependent; lower throughput.|Tracked automatically in LangSmith traces.|Failed to execute tool loop effectively for this prompt.|
|**Engineering Lesson**|Prompts tuned for local models remain reliable under strict guidelines.|Benchmarking is essential before updating model versions in production.|Newer or larger models do not guarantee better tool execution.|

## Important Definitions

|**Term**|**Definition**|**Why It Matters**|
|---|---|---|
|**`init_chat_model`**|A LangChain factory function that instantiates chat model objects dynamically using a string identifier.|Standardizes initialization across providers without importing model-specific classes.|
|**`bind_tools`**|A method that attaches a list of tools to an LLM instance by converting them to vendor-native JSON schemas.|Exposes function-calling options to the model during the reasoning pass.|
|**`@tool` Decorator**|A decorator that converts a standard Python function into a structured LangChain tool object.|Automatically extracts function names, docstrings, argument types, and return signatures into model schemas.|
|**`@traceable`**|A LangSmith decorator that traces a wrapped Python function's execution.|Aggregates individual sub-steps, token costs, and latencies into a unified trace.|
|**Defensive Prompting**|Structuring system messages with explicit constraints to reduce hallucinated tool arguments.|Essential for smaller open-weights models that may otherwise skip tool calls.|

## Code & Implementations

### Full Script: Custom ReAct Agent Loop with LangSmith Tracing

Below is the full implementation built with LangChain primitives, manual context appending, safety bounds, and Ollama integration.

Python

```python
import os
from typing import Dict, Any
from dotenv import load_dotenv
from langsmith import traceable

from langchain.chat_models import init_chat_model
from langchain_core.tools import tool
from langchain_core.messages import SystemMessage, HumanMessage, ToolMessage, AIMessage

# 1. Environment Loading & Tracing Setup
load_dotenv()

# Limit agent execution loop bounds
MAX_ITERATIONS = 10[cite: 13]
MODEL_NAME = "qwen3:1.7b"[cite: 13]

# 2. Define Custom Tools with Schema Metadata
@tool
def get_product_price(product_name: str) -> float:
    """Retrieves the price for a product in the e-commerce catalog."""
    print(f"\n[EXECUTION] get_product_price called with product_name='{product_name}'")
    catalog = {
        "laptop": 1299.00,
        "headphones": 199.50,
        "keyboard": 89.99
    }
    return catalog.get(product_name.lower(), 0.0)[cite: 13]

@tool
def apply_discount(price: float, tier: str) -> float:
    """Applies a discount tier to a given price and returns the updated price.
    Available tiers: bronze, silver, gold.
    """
    print(f"\n[EXECUTION] apply_discount called with price={price}, tier='{tier}'")
    discount_rates = {
        "bronze": 5,    # 5%
        "silver": 12,   # 12%
        "gold": 23      # 23%
    }[cite: 13]
    
    rate = discount_rates.get(tier.lower(), 0)
    final_price = price * (1.0 - (rate / 100.0))
    return round(final_price, 2)[cite: 13]

# 3. Custom Agent Loop Wrapped with LangSmith @traceable
@traceable(name="LangChain Agent Loop")
def run_agent(question: str) -> str:
    # Build tool registry lookup table
    tools = [get_product_price, apply_discount][cite: 12]
    tool_dict: Dict[str, Any] = {t.name: t for t in tools}[cite: 10, 12]

    # Initialize model and bind tools
    llm = init_chat_model(MODEL_NAME, model_provider="ollama")[cite: 12, 13]
    llm_with_tools = llm.bind_tools(tools)[cite: 12]

    # Defensive System Prompt Instructions
    system_prompt = SystemMessage(
        content=(
            "You are a helpful shopping assistant. "
            "You have access to a product catalog tool and a discount tool.\n"
            "STRICT RULES:\n"
            "1. Never guess or assume any product price.\n"
            "2. You MUST call the get_product_price tool to obtain the true catalog price.\n"
            "3. Only call apply_discount after obtaining a valid price from get_product_price.\n"
            "4. Never calculate discounts manually using math.\n"
            "5. If a discount tier is not specified by the user, ask for clarification."
        )[cite: 12]
    )

    # Initialize message list with system prompt and user input
    messages = [
        system_prompt,
        HumanMessage(content=question)
    ][cite: 10, 12]

    print(f"User Question: {question}\n" + "=" * 50)

    # 4. Explicit ReAct Execution Loop
    for iteration in range(1, MAX_ITERATIONS + 1):
        print(f"\n--- Iteration {iteration} ---")[cite: 10, 11]
        
        # Step A: Thought Step (LLM invocation)
        ai_message: AIMessage = llm_with_tools.invoke(messages)[cite: 10, 11]
        
        # Step B: Terminal Condition Check
        if not ai_message.tool_calls:
            print("\n[FINAL ANSWER OBTAINED]")
            print(f"Response: {ai_message.content}")
            return str(ai_message.content)[cite: 10, 11]

        # Process the primary requested tool call
        primary_tool_call = ai_message.tool_calls[0][cite: 10, 11]
        tool_name = primary_tool_call["name"][cite: 10, 11]
        tool_args = primary_tool_call["args"][cite: 10, 11]
        tool_call_id = primary_tool_call["id"][cite: 10, 11]

        print(f"[THOUGHT] Selected Tool: '{tool_name}'")
        print(f"[THOUGHT] Target Arguments: {tool_args}")

        # Step C: Execution Step
        target_function = tool_dict.get(tool_name)[cite: 10, 11]
        if not target_function:
            raise ValueError(f"Tool '{tool_name}' not found in registry.")[cite: 10, 11]

        observation = target_function.invoke(tool_args)[cite: 10, 11]
        print(f"[OBSERVATION] Result: {observation}")[cite: 10, 11]

        # Step D: Context Maintenance (Scratchpad Updates)
        messages.append(ai_message)[cite: 10, 11]  # Record reasoning and tool choice
        messages.append(
            ToolMessage(
                content=str(observation),
                tool_call_id=tool_call_id,
                name=tool_name
            )[cite: 10, 11]
        )  # Record execution observation

    print("\n[ERROR] Agent exceeded maximum iteration bounds.")[cite: 10, 11]
    return ""

if __name__ == "__main__":
    run_agent("What is the price of the laptop after applying the gold discount?")
```

## Visual Diagrams

### Iterative State Transformation Flow

Diagram showing how `messages` grow across iterations inside the ReAct loop:

```
INITIAL STATE:
┌──────────────────────────────────────────────────────────┐
│ [SystemMessage] Defensive Instructions                   │
│ [HumanMessage] "Price of laptop with gold discount?"     │
└──────────────────────────────────────────────────────────┘
                             │
                             ▼
ITERATION 1 STATE:
┌──────────────────────────────────────────────────────────┐
│ ... (Initial Messages)                                   │
│ [AIMessage] Tool Call: get_product_price(product='laptop')│
│ [ToolMessage] Observation: 1299.0                       │
└──────────────────────────────────────────────────────────┘
                             │
                             ▼
ITERATION 2 STATE:
┌──────────────────────────────────────────────────────────┐
│ ... (Iteration 1 Messages)                               │
│ [AIMessage] Tool Call: apply_discount(price=1299, tier='gold') │
│ [ToolMessage] Observation: 1000.23                       │
└──────────────────────────────────────────────────────────┘
                             │
                             ▼
ITERATION 3 STATE (FINAL):
┌──────────────────────────────────────────────────────────┐
│ ... (Iteration 2 Messages)                               │
│ [AIMessage] "The price of the laptop with gold discount is $1000.23" │
└──────────────────────────────────────────────────────────┘
```

## Common Pitfalls & Best Practices

### Mistakes to Avoid

> [!warning] **Unvalidated Model Swapping** Swapping model strings (e.g., changing from local Ollama to a newer OpenAI model) without evaluating performance can break agent behavior.

> [!warning] **Omitting Both Tool Calls and Responses** Modifying message histories manually without including both the `AIMessage` (containing the tool call request) and the matching `ToolMessage` (containing the execution result) will cause provider API errors.

> [!warning] **Missing Tool Call Identifier Alignment** Omitting the `tool_call_id` parameter when appending a `ToolMessage` prevents the LLM from associating execution observations with their original requests.

### Best Practices

> [!tip] **Strict Defensive System Prompting** When running smaller open-weights models (like `qwen3:1.7b`), add explicit rules to prevent the model from guessing or manually calculating values.

> [!tip] **Traceability Integration** Use the `@traceable` decorator from `langsmith` on entry-point functions to aggregate nested tool calls, token usage, and total latency within a single trace dashboard.

## Active Recall & Interview Prep

### Key Q&A Flashcards

**Q: Why is `init_chat_model` preferred over direct model imports?** **A:** It abstracts provider initialization into a single string-based interface, making it easy to swap models across supported backends.

**Q: How does the `@tool` decorator assist model execution?** **A:** It parses Python function names, type hints, and docstrings into model-agnostic JSON Schemas passed via `bind_tools`.

**Q: What indicates that a ReAct loop has reached its terminal condition?** **A:** The `AIMessage` returned by the model contains an empty `tool_calls` list, signaling that it has generated a final answer.

**Q: Why is defensive system prompting especially important for open-weights models?** **A:** Smaller models can hallucinate tool parameters or attempt mental math instead of issuing required tool calls.

**Q: Why must both an `AIMessage` and a `ToolMessage` be appended to the scratchpad after a tool call?** **A:** The `AIMessage` records the model's tool selection request, while the `ToolMessage` provides the execution result linked by `tool_call_id`.

**Q: Does upgrading to a newer LLM version guarantee better agent performance?** **A:** No. Model updates can alter instruction following or tool-calling behavior, making benchmarking necessary before upgrading.

### Practical Practice Scenario

**Scenario:** An e-commerce agent using a local LLM occasionally hallucinates product prices instead of invoking `get_product_price`. How can you resolve this issue in your code?

**Solution/Approach:**

1. **Apply Defensive System Prompting:** Add strict instructions in the `SystemMessage` explicitly forbidding price assumptions or manual math calculations.
    
2. **Verify Tool Binding:** Ensure `bind_tools` is called correctly on the initialized chat model.
    
3. **Enhance Function Docstrings:** Write explicit docstrings for `@tool` definitions, detailing required input parameters and expected return values.
    

## One-Page Cheat Sheet

- **ReAct Core:** Iterative Thought $\rightarrow$ Action $\rightarrow$ Observation state loop.
    
- **`init_chat_model`:** Initializing chat models using string identifiers.
    
- **`@tool` Decorator:** Converts Python functions into structured tool schemas.
    
- **`bind_tools`:** Attaches tools to language model instances.
    
- **Terminal Condition:** Empty `tool_calls` array in the returned `AIMessage`.
    
- **Context Maintenance:** Requires appending both the `AIMessage` and matching `ToolMessage`.
    
- **Trace Alignment:** Link `ToolMessage` entries using the `tool_call_id` from the model's request.
    
- **Defensive Prompting:** Use strict system rules to prevent smaller models from guessing values.
    
- **Benchmarking Requirement:** Test models on target tasks before swapping versions in production.
    
- **`@traceable` Decorator:** Groups agent loops and nested sub-calls within LangSmith dashboards.
    
- **Iteration Limits:** Enforce maximum loop bounds (e.g., `MAX_ITERATIONS = 10`) to prevent infinite execution.