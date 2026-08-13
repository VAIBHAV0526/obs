# LangChain: Agentic AI Engineering — ReAct Loop & Under-the-Hood Architecture

## Metadata

- **Topic:** AI Agent Architecture, ReAct Loop Mechanics, Function Calling, and Framework Abstractions
    
- **Difficulty:** Intermediate / Advanced
    
- **Tags:** #langchain #ollama #react-loop #agentic-ai #python #llm-architecture #system-design
    
- **Source:** Agentic AI Engineering with LangChain & LangGraph
    
- **Date:** 2026-08-05
    

## Executive Summary

- **The Core Agentic Loop:** Modern autonomous AI agents (e.g., Devin, Claude Code) rely on the ReAct (Reasoning + Acting) execution pattern—a continuously running `while` loop that iterates until an explicit terminal condition is met.
    
- **The Three-Step Cycle:** Each agent iteration moves sequentially through **Thought** (LLM reasoning/tool selection), **Action** (application tool execution), and **Observation** (returning execution results back to context).
    
- **The Scratchpad Pattern:** The agent preserves state by appending all intermediate tool calls, execution payloads, and context histories into a running conversation trace known as the scratchpad.
    
- **Peeling Abstractions (Layers 0 to 3):** Agent engineering evolves from high-level abstractions (`create_agent`) down to primitive tool bindings, raw JSON-Schema model API payloads, and pure ReAct text prompts using Regex parsing.
    
- **Local First Development with Ollama & `uv`:** Python environments can be managed via `uv`, leveraging open-weights models (such as `qwen3:1.7b`) running locally on Ollama alongside LangSmith execution tracing.
    

## Main Concepts & Theory

### 1. The ReAct Architecture

First introduced in 2023 by Princeton University and Google Research engineers, the ReAct pattern combines reasoning (Thought) and action execution (Act) to perform complex multi-step tasks.

```mermaid
                          ┌───────────────────────────────┐
                          │          User Query           │
                          └───────────────┬───────────────┘
                                          │
                                          ▼
                         ┌─────────────────────────────────┐
  ┌─────────────────────►│         1. THOUGHT (LLM)        │
  │                      │ Evaluates prompt + scratchpad;   │
  │                      │ decides next tool or final answer│
  │                      └────────────────┬────────────────┘
  │                                       │
  │                               Requires Tool?
  │                               /               \
  │                         YES  /                 \  NO (Final Answer)
  │                             /                   \
  │                            ▼                     ▼
  │             ┌──────────────────────────┐    ┌──────────────────────────┐
  │             │     2. ACTION (App)      │    │      Return Output       │
  │             │ Executes application tool│    │       to User            │
  │             └─────────────┬────────────┘    └──────────────────────────┘
  │                           │
  │                           ▼
  │             ┌──────────────────────────┐
  │             │ 3. OBSERVATION (Context) │
  │             │ Appends tool output to   │
  │             │ scratchpad/message trace │
  │             └─────────────┬────────────┘
  │                           │
  └───────────────────────────┘
```

#### The ReAct Iteration Loop Breakdown

1. **Thought (Reasoning):** The LLM processes the user prompt, system instructions, available tool definitions, and historical scratchpad context. It decides whether to emit a function execution request or complete the request.
    
2. **Action (Execution):** If a tool call is chosen, the agent runtime interrupts LLM generation, extracts the function name and structured arguments, and executes the physical Python tool within the host application environment.
    
3. **Observation (Feedback):** The output from the application function is formatted into a message object (e.g., `ToolMessage`) and appended to the context window. The loop then re-prompts the LLM with this expanded history.
    

### 2. Layers of Agent Abstraction

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Layer 0: High-Level Abstractions (create_agent)                        │
├─────────────────────────────────────────────────────────────────────────┤
│ Layer 1: Framework Primitives (LangChain bind_tools, ToolMessage)       │
├─────────────────────────────────────────────────────────────────────────┤
│ Layer 2: Raw Vendor APIs & Manual JSON Schema Definitions               │
├─────────────────────────────────────────────────────────────────────────┤
│ Layer 3: Native ReAct Prompts + Regex Parsing (No Function Calling)     │
└─────────────────────────────────────────────────────────────────────────┘
```

|**Layer**|**Implementation Style**|**Mechanism**|**Advantages / Key Learning**|
|---|---|---|---|
|**Layer 0**|High-level framework|Standard helper functions (`create_agent`) manage the state loop automatically.|Maximum developer velocity; obscures loop mechanics.|
|**Layer 1**|Framework Primitives|Uses LangChain tools, model bindings (`bind_tools`), and structured messages (`ToolMessage`) inside a custom `while` loop.|Eliminates API boilerplate while retaining explicit loop control.|
|**Layer 2**|Raw Vendor APIs|Direct HTTP/SDK calls passing explicit JSON Schemas to vendor function-calling endpoints.|Exposes model-agnostic schema translation and vendor boundaries.|
|**Layer 3**|Text Prompts & Regex|Raw prompt engineering (Thought/Action/Observation text format) parsed via Regular Expressions.|Demonstrates how agents functioned prior to native LLM tool calling APIs.|

## Important Definitions

|**Term**|**Definition**|**Why It Matters**|
|---|---|---|
|**ReAct Loop**|An execution pattern where an LLM alternates between reasoning ("Thought") and tool execution ("Action") until a goal is achieved.|Forms the core architectural foundation for modern autonomous AI software agents.|
|**Scratchpad**|The cumulative message history storing past inputs, system prompts, LLM tool requests, and resulting execution outputs.|Ensures the LLM retains full context of prior actions across iterative loop passes.|
|**Thought**|The internal reasoning phase where the LLM determines its next execution step based on available information.|Prevents premature or misdirected tool calls by forcing contextual evaluation.|
|**Action**|The formal request emitted by the LLM specifying a target function name and input parameters.|Converts model text output into deterministic software execution.|
|**Observation**|The structured raw data returned by an executed application tool back to the LLM.|Grounds the LLM's next reasoning step in actual real-time execution results.|

## Code & Implementations

### Environment & Tooling Setup

Environment initialization script using `uv`, local Ollama model management, and LangSmith environment configurations:

Bash

```
# Initialize local Python environment
uv init
rm main.py

# Install essential dependencies
uv add langchain langchain-ollama langchain-openai python-dotenv black isort

# Pull local open-weights model supporting function calling
ollama pull qwen3:1.7b

# Verify and execute local Ollama server
ollama serve
```

Ini, TOML

```
# Environment variables (.env)
OPENAI_API_KEY=sk-proj-xxxx...
LANGSMITH_API_KEY=lsv2_pt_xxxx...
LANGSMITH_TRACING=true
LANGSMITH_PROJECT=ReAct Under The Hood
```

### Layer 1: Custom ReAct Agent Loop with LangChain Primitives

Building an explicit agent execution loop for an e-commerce pricing and discount engine using LangChain primitives:

Python

```
import os
from dotenv import load_dotenv
from langchain_core.tools import tool
from langchain_core.messages import HumanMessage, AIMessage, ToolMessage
from langchain_ollama import ChatOllama

load_dotenv()

# --- 1. Define Tools ---
@tool
def get_product_price(product_name: str) -> float:
    """Retrieves the standard catalog price for a given product."""
    catalog = {
        "laptop": 1200.0,
        "headphones": 150.0,
        "keyboard": 80.0
    }
    return catalog.get(product_name.lower(), 0.0)

@tool
def apply_discount(price: float, tier: str) -> float:
    """Calculates final price after applying discount tier percentage."""
    tiers = {
        "bronze": 0.05,  # 5% off
        "silver": 0.10,  # 10% off
        "gold": 0.15     # 15% off
    }
    discount = tiers.get(tier.lower(), 0.0)
    return price * (1.0 - discount)

# Map tools for dynamic application-side invocation
tools = [get_product_price, apply_discount]
tools_by_name = {t.name: t for t in tools}

# --- 2. Initialize Model with Tool Bindings ---
llm = ChatOllama(model="qwen3:1.7b", temperature=0)
llm_with_tools = llm.bind_tools(tools)

# --- 3. Custom ReAct Agent Loop ---
def run_agent(user_query: str):
    # Initialize message history (Scratchpad context)
    messages = [HumanMessage(content=user_query)]
    
    print(f"User Query: {user_query}\n" + "="*50)
    
    while True:
        # Step 1: THOUGHT (LLM Evaluation)
        response = llm_with_tools.invoke(messages)
        messages.append(response)
        
        # Terminal condition: If no tool calls are requested, LLM has provided final answer
        if not response.tool_calls:
            print(f"\nFinal Answer: {response.content}")
            return response.content
        
        # Step 2: ACTION & OBSERVATION Loop
        for tool_call in response.tool_calls:
            tool_name = tool_call["name"]
            tool_args = tool_call["args"]
            tool_call_id = tool_call["id"]
            
            print(f"[ACTION] Calling tool '{tool_name}' with args: {tool_args}")
            
            # Application execution
            target_tool = tools_by_name[tool_name]
            observation = target_tool.invoke(tool_args)
            
            print(f"[OBSERVATION] Tool output: {observation}")
            
            # Append Observation back into Scratchpad context
            messages.append(
                ToolMessage(
                    content=str(observation),
                    tool_call_id=tool_call_id,
                    name=tool_name
                )
            )

# Run Example
if __name__ == "__main__":
    run_agent("What is the final price of a laptop with a gold discount?")
```

## Visual Diagrams

### Detailed Message Trace Flow inside the ReAct Loop

Code snippet

```
sequenceDiagram
    autonumber
    actor User
    participant Loop as Python Agent Loop
    participant LLM as Model (Qwen3 / OpenAI)
    participant Tools as Local Python Functions

    User->>Loop: Query: "Laptop price with gold discount?"
    Note over Loop: Init Scratchpad: [HumanMessage]

    rect rgb(235, 245, 255)
        Note over Loop,LLM: Iteration 1
        Loop->>LLM: Pass Scratchpad Context
        LLM-->>Loop: AIMessage (Tool Call: get_product_price(product_name="laptop"))
        Loop->>Tools: Execute get_product_price("laptop")
        Tools-->>Loop: Return 1200.0
        Note over Loop: Append ToolMessage(1200.0) to Scratchpad
    end

    rect rgb(235, 255, 235)
        Note over Loop,LLM: Iteration 2
        Loop->>LLM: Pass Updated Scratchpad Context
        LLM-->>Loop: AIMessage (Tool Call: apply_discount(price=1200.0, tier="gold"))
        Loop->>Tools: Execute apply_discount(1200.0, "gold")
        Tools-->>Loop: Return 1020.0
        Note over Loop: Append ToolMessage(1020.0) to Scratchpad
    end

    rect rgb(255, 245, 235)
        Note over Loop,LLM: Iteration 3
        Loop->>LLM: Pass Complete Scratchpad Context
        LLM-->>Loop: AIMessage ("The final price for a laptop with gold discount is $1020.00.")
    end

    Loop-->>User: Output Final Answer
```

## System Architecture & Trade-offs

### System Trade-offs Across Abstraction Layers

```
Layer 0 (create_agent)          ──────► Maximum Convenience / Low Transparency
Layer 1 (LangChain Primitives)  ──────► Balanced / Explicit Control & Safety
Layer 2 (Raw Model APIs)        ──────► Framework Independence / Verbose Setup
Layer 3 (Regex ReAct Prompts)   ──────► Maximum Transparency / Higher Fragility
```

#### Abstraction Layer Trade-offs

|**Dimension**|**High-Level Helper (create_agent)**|**Explicit Framework Loop (Layer 1)**|**Raw ReAct Prompting (Layer 3)**|
|---|---|---|---|
|**Development Speed**|Immediate out-of-the-box prototype creation.|Moderate setup overhead requiring simple explicit `while` loops.|High engineering overhead requiring custom string parsers.|
|**Execution Transparency**|Black-box execution; difficult to customize intermediate steps.|High visibility into message transitions and tool calls.|Complete control over every raw token sent and received.|
|**Parsing Reliability**|High (handled via vendor function calling APIs).|High (schema enforcement via standard models).|Low (vulnerable to LLM prompt formatting drifts and Regex breaks).|
|**Model Flexibility**|Seamless model switching across unified provider APIs.|Easy swapping of underlying chat model abstractions.|Works on older/legacy models lacking native function-calling APIs.|

## Common Pitfalls & Best Practices

### Mistakes to Avoid

> [!warning] **Infinite ReAct Execution Loops** Uncapped `while True:` loops can run indefinitely if an LLM gets trapped in repetitive tool invocations. Always implement an explicit iteration counter check (`max_iterations = 10`).

> [!warning] **Omitting Tool Execution Output from History** Failing to append `ToolMessage` or observation context back into the conversation array breaks the ReAct pattern, causing the LLM to call the same tool repeatedly without progressing.

> [!warning] **Over-reliance on Native Text Extraction for Tool Arguments** Manually parsing tool parameters using regex strings from raw model outputs is fragile compared to using native JSON schema validation interfaces.

### Best Practices

> [!tip] **Isolate Application Tools from Framework Logic** Write standard Python functions using explicit type hints and clean docstrings. Bind them to agent runtimes using framework-provided decorators (`@tool`) to generate accurate JSON Schemas.

> [!tip] **Enable Granular Tracing in Local Development** Configure tracing options (such as `LANGSMITH_TRACING=true`) when running local open-weights models through Ollama. This allows visual inspection of intermediate reasoning tokens and individual tool call payloads.

## Active Recall & Interview Prep

### Key Q&A Flashcards

**Q: What are the three fundamental phases of the ReAct execution pattern?** **A:** **Thought** (LLM reasoning and decision step), **Action** (calling specific application tools), and **Observation** (returning execution results back to context).

**Q: What condition causes a ReAct agent loop to break and return an answer?** **A:** The loop terminates when the LLM generates a response that contains no function or tool call requests, signaling that it has enough information to formulate a final text answer.

**Q: What role does the "scratchpad" play in an autonomous agent architecture?** **A:** It maintains the cumulative state and history of all user prompts, LLM tool invocation requests, and returned tool output observations across loop iterations.

**Q: Why is `uv` preferred for managing local agentic Python environments?** **A:** `uv` provides fast project initialization (`uv init`), dependency locking (`uv.lock`), and isolated package execution for complex AI dependencies.

**Q: What is the primary difference between Layer 1 and Layer 3 agent implementations?** **A:** Layer 1 uses framework primitives (like `bind_tools` and structured `ToolMessage` objects) relying on native model function calling APIs. Layer 3 uses raw string prompts and regex parsing to execute tool calls manually.

**Q: How does an agent runtime pass a tool's output back to the language model?** **A:** The tool's output string is wrapped inside a structured observation payload (e.g., `ToolMessage`) tagged with the originating `tool_call_id` and appended to the history array.

### Practical Practice Scenario

**Scenario:** You are building an agent that calculates total employee compensation. The model needs to call `get_base_salary(employee_id)` and then `calculate_bonus(base_salary, performance_rating)`. During testing, the agent calls `get_base_salary` repeatedly in an infinite loop.

**Solution/Approach:**

1. **Inspect Context Appends:** Verify that the observation output from `get_base_salary` is being appended to the context list as a properly tagged `ToolMessage` containing the matching `tool_call_id`.
    
2. **Add Maximum Iteration Guardrail:** Introduce a loop safety limit to break execution if iteration count exceeds a threshold:
    

Python

```
max_iterations = 5
iteration = 0

while iteration < max_iterations:
    iteration += 1
    # Agent execution logic...
```

3. **Verify Tool Schema Descriptions:** Ensure docstrings and argument types explicitly describe inputs so the LLM realizes the first step has completed and can proceed to call `calculate_bonus`.
    

## One-Page Cheat Sheet

- **ReAct Core:** Thought $\rightarrow$ Action $\rightarrow$ Observation loop.
    
- **Thought Phase:** LLM processes scratchpad context to select a tool or formulate a final response.
    
- **Action Phase:** Application runtime extracts tool arguments and executes local functions.
    
- **Observation Phase:** Raw tool execution results are appended back into context.
    
- **Scratchpad Context:** Persistent array of all messages, inputs, tool calls, and outputs.
    
- **Loop Terminal Condition:** LLM returns a payload with zero requested `tool_calls`.
    
- **Abstraction Peeling:** Layer 0 (`create_agent`) $\rightarrow$ Layer 1 (Primitives) $\rightarrow$ Layer 2 (Raw Vendor APIs) $\rightarrow$ Layer 3 (Regex String Prompts).
    
- **Local Agent Tooling:** Run lightweight models locally via `ollama run qwen3:1.7b`.
    
- **Environment Execution:** Use `uv init` and `uv add` for managing dependencies.
    
- **Tool Bindings:** Use `llm.bind_tools([...])` to automatically attach JSON tool schemas to chat models.
    
- **`ToolMessage` Object:** Stores observation strings linked directly to specific `tool_call_id` keys.
    
- **Safety Rules:** Always enforce `max_iterations` boundaries on raw `while True:` execution loops.