# Top 3 Python Packages Every Agentic SDK Developer Must Know

Whether you're building with **Claude Agent SDK**, **OpenAI Agents SDK**, **LangChain/LangGraph**, **CrewAI**, **AutoGen**, or **LlamaIndex** — three Python packages keep showing up everywhere. They're not SDK-specific. They're the foundation underneath all of them.

This post breaks down what each one does, why it matters, and how to use it effectively in agentic workflows.

---

![Agentic Stack Overview](/images/agentic-stack-overview.svg)

---

## The Three Packages

| Package | Role | Install |
|---|---|---|
| `pydantic` | Structured data, tool schemas, validation | `pip install pydantic` |
| `asyncio` | Concurrency engine — event loop, gather, tasks | stdlib — no install |
| `tenacity` | Retry logic and resilience for LLM calls | `pip install tenacity` |

---

## 1. Pydantic — Structured Everything

**Why it matters:** LLMs communicate via text. Pydantic converts that unstructured text into typed, validated Python objects — and converts your Python objects into the JSON schemas LLMs need to call your tools correctly.

![Pydantic Flow](/images/pydantic-flow.svg)

### Use Case 1: Tool / Function Schemas

Every agentic SDK needs to describe tools to the LLM. Pydantic `BaseModel` auto-generates the JSON schema the LLM uses to fill arguments correctly.

```python
from pydantic import BaseModel, Field
from typing import Literal

class WebSearchInput(BaseModel):
    query: str = Field(description="Search query string")
    max_results: int = Field(default=5, ge=1, le=20)
    language: str = Field(default="en")

# Pydantic generates this JSON schema automatically:
# {
#   "properties": {
#     "query": {"type": "string", "description": "Search query string"},
#     "max_results": {"type": "integer", "default": 5, "minimum": 1, "maximum": 20},
#     "language": {"type": "string", "default": "en"}
#   },
#   "required": ["query"]
# }
```

No schema → the LLM guesses argument names → broken tool calls.

**How it maps across SDKs:**

```python
# LangChain
from langchain.tools import StructuredTool
tool = StructuredTool.from_function(func=search, args_schema=WebSearchInput)

# OpenAI Agents SDK
from agents import function_tool
@function_tool
def search(input: WebSearchInput) -> str: ...

# Claude Agent SDK
# Tool schema derived from Pydantic model's .model_json_schema()

# CrewAI
from crewai.tools import BaseTool
class SearchTool(BaseTool):
    args_schema: Type[BaseModel] = WebSearchInput
```

### Use Case 2: Structured LLM Outputs

Instead of parsing LLM responses with regex or string splits, force the model to return a specific structure.

```python
from pydantic import BaseModel
from typing import Literal, List

class ResearchSummary(BaseModel):
    title: str
    key_findings: List[str]
    sentiment: Literal["positive", "negative", "neutral"]
    confidence: float
    sources: List[str]

# LangChain
chain = prompt | llm.with_structured_output(ResearchSummary)
result: ResearchSummary = chain.invoke({"topic": "AI agents"})
print(result.confidence)  # float, guaranteed

# OpenAI SDK
response = client.beta.chat.completions.parse(
    model="gpt-4o",
    messages=[...],
    response_format=ResearchSummary,
)
result: ResearchSummary = response.choices[0].message.parsed
```

The LLM is constrained to produce JSON that matches the schema. No more `"I think the sentiment is positive"` — you get `{"sentiment": "positive"}`, always.

### Use Case 3: Agent State Models

LangGraph and other graph-based agents pass state between nodes. Pydantic keeps that state typed and validated.

```python
from pydantic import BaseModel
from typing import List, Optional, Annotated
import operator

class Message(BaseModel):
    role: Literal["user", "assistant", "tool"]
    content: str

class AgentState(BaseModel):
    messages: Annotated[List[Message], operator.add]  # LangGraph reducer
    current_step: int = 0
    task_complete: bool = False
    error_count: int = 0
    final_answer: Optional[str] = None

def research_node(state: AgentState) -> AgentState:
    # state.messages is guaranteed to be List[Message]
    # state.current_step is guaranteed to be int
    ...
```

### Pydantic v2 Performance Note

Pydantic v2 rewrote the core in Rust. Validation is 5–50x faster than v1. Most SDKs now require v2.

```python
import pydantic; print(pydantic.__version__)  # should be 2.x

from pydantic import field_validator

class AgentConfig(BaseModel):
    max_steps: int = 10
    temperature: float = 0.7

    @field_validator("temperature")
    @classmethod
    def validate_temperature(cls, v: float) -> float:
        if not 0 <= v <= 2:
            raise ValueError("temperature must be between 0 and 2")
        return v
```

---

## 2. asyncio — The Concurrency Engine

**Why it matters:** Agentic SDKs are async at the core. `asyncio` is the Python stdlib that makes concurrent agent execution possible — running multiple tool calls simultaneously, managing background tasks, and keeping the event loop unblocked. Every SDK is built on top of it.

![asyncio Flow](/images/asyncio-flow.svg)

### The Event Loop — What Actually Runs Your Agent

```python
import asyncio

# Every async agent entry point looks like this
async def run_agent():
    result = await agent.run("Research quantum computing trends")
    print(result)

# asyncio runs the event loop
asyncio.run(run_agent())
```

The event loop is a single thread that switches between coroutines whenever one is waiting on I/O (an LLM call, a tool call, a database query). This means hundreds of concurrent operations with no threading overhead.

### asyncio.gather — Run Tools Concurrently

The most important pattern in agentic systems. When an agent needs to call multiple tools, run them all at once:

```python
import asyncio

async def research_step(query: str) -> dict:
    # Without gather — sequential, slow
    # r1 = await search_web(query)      # waits 1s
    # r2 = await search_news(query)     # waits 1s after r1
    # r3 = await search_papers(query)   # waits 1s after r2
    # Total: ~3s

    # With gather — concurrent, fast
    r1, r2, r3 = await asyncio.gather(
        search_web(query),
        search_news(query),
        search_papers(query),
        return_exceptions=True,  # one failure doesn't kill all
    )
    # Total: ~1s (all run in parallel)

    results = {}
    for name, r in [("web", r1), ("news", r2), ("papers", r3)]:
        if not isinstance(r, Exception):
            results[name] = r
    return results
```

**The `return_exceptions=True` flag is critical.** Without it, one failing tool call cancels all the others.

### asyncio.create_task — Background Tasks

Fire non-critical work in the background while the agent continues:

```python
import asyncio

async def agent_step(state: AgentState) -> AgentState:
    # Kick off observability logging in background
    # Agent doesn't wait for this
    asyncio.create_task(log_step_to_observability(state))
    asyncio.create_task(update_progress_ui(state.current_step))

    # Agent proceeds immediately with main work
    llm_response = await call_llm(state.messages)
    return state.model_copy(update={"messages": state.messages + [llm_response]})
```

### asyncio.timeout — Guard Slow Tools

Prevent one slow tool call from stalling the entire agent:

```python
import asyncio

async def safe_tool_call(query: str) -> str | None:
    try:
        async with asyncio.timeout(10.0):  # Python 3.11+
            return await slow_external_tool(query)
    except asyncio.TimeoutError:
        return None  # tool timed out, agent continues

# Pre-3.11 equivalent:
result = await asyncio.wait_for(slow_external_tool(query), timeout=10.0)
```

### asyncio.Queue — Agent Message Bus

For multi-agent systems where agents need to pass tasks to each other:

```python
import asyncio

async def orchestrator(task_queue: asyncio.Queue, result_queue: asyncio.Queue):
    """Dispatches tasks to worker agents."""
    tasks = ["research X", "research Y", "research Z"]
    for task in tasks:
        await task_queue.put(task)

async def worker_agent(task_queue: asyncio.Queue, result_queue: asyncio.Queue):
    """Picks up tasks, executes them, returns results."""
    while True:
        task = await task_queue.get()
        result = await execute_task(task)
        await result_queue.put(result)
        task_queue.task_done()

async def run_multi_agent():
    task_q = asyncio.Queue()
    result_q = asyncio.Queue()

    # Run orchestrator + 3 workers concurrently
    await asyncio.gather(
        orchestrator(task_q, result_q),
        worker_agent(task_q, result_q),
        worker_agent(task_q, result_q),
        worker_agent(task_q, result_q),
    )
```

### asyncio in LangGraph

LangGraph nodes are async coroutines. The graph runner is the event loop:

```python
from langgraph.graph import StateGraph
from typing import TypedDict

class State(TypedDict):
    messages: list
    results: dict

# Each node is an async function
async def research_node(state: State) -> State:
    results = await asyncio.gather(
        search_web(state["messages"][-1]),
        search_news(state["messages"][-1]),
    )
    return {"results": {"web": results[0], "news": results[1]}}

async def analysis_node(state: State) -> State:
    analysis = await call_llm(f"Analyze: {state['results']}")
    return {"messages": state["messages"] + [analysis]}

graph = StateGraph(State)
graph.add_node("research", research_node)
graph.add_node("analysis", analysis_node)
graph.add_edge("research", "analysis")
app = graph.compile()

# asyncio runs the whole graph
result = await app.ainvoke({"messages": ["Research AI trends"], "results": {}})
```

---

## 3. tenacity — Retry Logic for Production Agents

**Why it matters:** LLM APIs fail. Rate limits happen. Network blips occur. Without retry logic, your agent crashes on a transient error. With tenacity, one decorator handles all of it.

![tenacity Retry Flow](/images/tenacity-retry.svg)

### The Problem Without Retry Logic

```python
# Crashes on any API hiccup
async def call_llm(prompt: str) -> str:
    response = await client.messages.create(
        model="claude-opus-4-6",
        messages=[{"role": "user", "content": prompt}]
    )
    return response.content[0].text

# Rate limit hit? → RateLimitError → agent dead
# Network blip?  → ConnectionError → agent dead
# API timeout?   → TimeoutError → agent dead
```

### Basic Retry with tenacity

```python
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    retry_if_exception_type,
    before_sleep_log,
)
import logging
import anthropic

logger = logging.getLogger(__name__)

@retry(
    wait=wait_exponential(multiplier=1, min=1, max=60),
    stop=stop_after_attempt(4),
    retry=retry_if_exception_type((
        anthropic.RateLimitError,
        anthropic.APIConnectionError,
        anthropic.APITimeoutError,
    )),
    before_sleep=before_sleep_log(logger, logging.WARNING),
)
async def call_llm(prompt: str) -> str:
    response = await client.messages.create(
        model="claude-opus-4-6",
        max_tokens=1024,
        messages=[{"role": "user", "content": prompt}]
    )
    return response.content[0].text
```

**What happens on failure:**
- Attempt 1 fails → wait 1s → retry
- Attempt 2 fails → wait 2s → retry
- Attempt 3 fails → wait 4s → retry
- Attempt 4 fails → raise `RetryError`

### Retry for Tool Calls

Wrap tool calls independently so one flaky tool doesn't kill the entire agent:

```python
from tenacity import retry, stop_after_attempt, wait_fixed, retry_if_exception_type
import aiohttp

@retry(
    wait=wait_fixed(2),
    stop=stop_after_attempt(3),
    retry=retry_if_exception_type((aiohttp.ClientError, asyncio.TimeoutError)),
)
async def fetch_stock_price(ticker: str) -> float:
    async with aiohttp.ClientSession() as session:
        async with session.get(f"https://finance.api.com/quote/{ticker}") as r:
            data = await r.json()
            return data["price"]
```

### Custom Retry Conditions

Retry based on response content, not just exceptions:

```python
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_result

def is_incomplete_response(result: str) -> bool:
    if not result or len(result.strip()) < 10:
        return True
    if result.strip().endswith(("...", "—")):
        return True
    return False

@retry(
    wait=wait_exponential(min=1, max=30),
    stop=stop_after_attempt(3),
    retry=retry_if_result(is_incomplete_response),
)
async def get_complete_response(prompt: str) -> str:
    response = await client.messages.create(
        model="claude-opus-4-6",
        max_tokens=2048,
        messages=[{"role": "user", "content": prompt}]
    )
    return response.content[0].text
```

### Cross-SDK Retry Patterns

```python
# Anthropic
from anthropic import RateLimitError, APIConnectionError, APITimeoutError

@retry(
    wait=wait_exponential(min=1, max=60),
    stop=stop_after_attempt(4),
    retry=retry_if_exception_type((RateLimitError, APIConnectionError, APITimeoutError)),
)
async def anthropic_call(prompt: str): ...

# OpenAI
from openai import RateLimitError as OpenAIRateLimit, APIConnectionError

@retry(
    wait=wait_exponential(min=1, max=60),
    stop=stop_after_attempt(5),
    retry=retry_if_exception_type((OpenAIRateLimit, APIConnectionError)),
)
async def openai_call(prompt: str): ...

# LangChain built-in retry
chain_with_retry = chain.with_retry(
    retry_if_exception_type=(Exception,),
    stop_after_attempt=3,
    wait_exponential_jitter=True,
)
```

### Adding Jitter

Jitter prevents the "thundering herd" — multiple agents all retrying at the same moment and spiking the API:

```python
from tenacity import wait_random_exponential

@retry(
    wait=wait_random_exponential(min=1, max=60),  # randomized backoff
    stop=stop_after_attempt(6),
)
async def resilient_llm_call(prompt: str) -> str: ...
```

---

## Putting All Three Together

A production-ready agentic research step combining all three:

```python
import asyncio
from pydantic import BaseModel, Field
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type
from typing import List
import anthropic

client = anthropic.AsyncAnthropic()

# Pydantic: typed contracts
class ResearchQuery(BaseModel):
    topic: str = Field(description="Research topic")
    depth: int = Field(default=3, ge=1, le=5)

class ResearchResult(BaseModel):
    topic: str
    findings: List[str]
    confidence: float

# tenacity: resilient LLM call
@retry(
    wait=wait_exponential(min=1, max=60),
    stop=stop_after_attempt(4),
    retry=retry_if_exception_type((
        anthropic.RateLimitError,
        anthropic.APIConnectionError,
    )),
)
async def _call_llm(prompt: str) -> str:
    response = await client.messages.create(
        model="claude-opus-4-6",
        max_tokens=2048,
        messages=[{"role": "user", "content": prompt}],
    )
    return response.content[0].text

# asyncio: concurrent sub-queries
async def research(query: ResearchQuery) -> ResearchResult:
    sub_queries = [
        f"What is {query.topic}?",
        f"Latest developments in {query.topic}",
        f"Key challenges in {query.topic}",
    ]

    # asyncio.gather: all LLM calls run concurrently
    # tenacity: each one retries independently on failure
    responses = await asyncio.gather(
        *[_call_llm(q) for q in sub_queries],
        return_exceptions=True,
    )

    findings = [r for r in responses if not isinstance(r, Exception)]

    # Pydantic: validated, typed output
    return ResearchResult(
        topic=query.topic,
        findings=findings,
        confidence=len(findings) / len(sub_queries),
    )
```

---

## Quick Reference

```bash
pip install pydantic tenacity
# asyncio is built into Python — no install needed
```

| Package | Key APIs | When to Use |
|---|---|---|
| `pydantic` | `BaseModel`, `Field`, `model_validator` | Any data in/out of an agent or tool |
| `asyncio` | `gather`, `create_task`, `timeout`, `Queue` | Concurrent tools, event loop, background tasks |
| `tenacity` | `@retry`, `wait_exponential`, `stop_after_attempt` | Any LLM or API call that can fail |

These three underpin every major agentic SDK. Learn them once, use them everywhere.

---

## Summary

- **Pydantic** is the contract layer — typed schemas in, validated data out. LLMs know what to call and how.
- **asyncio** is the concurrency engine — runs multiple tools simultaneously, keeps the event loop unblocked, and is the foundation every async SDK is built on.
- **tenacity** is the resilience layer — wraps LLM and tool calls with retry logic so transient failures don't kill your agent.

Together they form the backbone of any production agentic system, regardless of which SDK you choose.
