# Langgraph
# LangGraph Agents: From Basic Chatbot to Human-in-the-Loop

Hands-on Jupyter notebooks that build up a LangGraph agent step by step: a plain chatbot, tool calling, a ReAct-style loop, persistent memory, streaming, and human-in-the-loop interrupts.

**Stack:** [LangGraph](https://langchain-ai.github.io/langgraph/) · [LangChain](https://python.langchain.com/) · [Groq](https://groq.com/) (`qwen/qwen3.8-27b`) · [Tavily Search](https://tavily.com/) · Python 3.12

---

## Notebooks

| Notebook | What it covers |
|---|---|
| [`1_basicchatbot.ipynb`](./1_basicchatbot.ipynb) | Basic chatbot, tool calling, ReAct agent, memory, streaming |
| [`Human_in_the_loop.ipynb`](./Human_in_the_loop.ipynb) | Pausing a graph with `interrupt()` and resuming with `Command` |

### 1. `1_basicchatbot.ipynb`

1. **Basic chatbot (Graph API):** `State` with the `add_messages` reducer, a single LLM node, `START → llmchatbot → END`.
2. **Chatbot with tools:** binds `TavilySearch` and a custom `multiply` function to the LLM; `ToolNode` + `tools_condition` route between the model and tools.
3. **ReAct agent architecture:** adds the `tools → tool_calling_llm` edge so the model can chain multiple tool calls (e.g. search, then multiply).
4. **Memory:** compares a stateless graph with one compiled using `MemorySaver` and a `thread_id`, so the bot remembers earlier turns.
5. **Streaming:** `.stream()` / `.astream()` with `stream_mode="values"` (full state after each node) and `"updates"` (only the state changes).

### 2. `Human_in_the_loop.ipynb`

Defines a `human_assistance` tool that calls `interrupt()`, pausing the graph until a human supplies an answer. Execution is then resumed with `Command(resume={"data": ...})`. Requires a checkpointer (`MemorySaver`) so the paused state can be restored.

---

## Graph Architecture

```
            ┌───────────┐   tool call    ┌─────────┐
 START ───▶ │  chatbot  │ ─────────────▶ │  tools  │
            │   (LLM)   │ ◀───────────── │         │
            └─────┬─────┘     result     └─────────┘
                  │ no tool call
                  ▼
                 END
```

In the human-in-the-loop notebook, `tools` includes both `TavilySearch` and `human_assistance`.

---

## Getting Started

### Prerequisites

- Python 3.12+
- A [Groq API key](https://console.groq.com/keys)
- A [Tavily API key](https://app.tavily.com/)

### Install

With [uv](https://docs.astral.sh/uv/):

```bash
git clone <your-repo-url>
cd <your-repo-name>

uv venv
.venv\Scripts\activate          # Windows
# source .venv/bin/activate     # macOS / Linux

uv pip install langgraph langchain langchain-groq langchain-tavily python-dotenv ipykernel
```

Or with pip:

```bash
python -m venv .venv
pip install langgraph langchain langchain-groq langchain-tavily python-dotenv ipykernel
```

### Configure environment variables

Create a `.env` file in the project root (and make sure it is in `.gitignore`):

```env
GROQ_API_KEY=your_groq_api_key
TAVILY_API_KEY=your_tavily_api_key
```

### Run

Open the notebooks in VS Code or Jupyter, select the `.venv` kernel, and run the cells top to bottom:

```bash
jupyter lab
```

---

## Human-in-the-Loop: Minimal Working Pattern

The graph must be **compiled with a checkpointer** for `interrupt()` to work:

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.types import Command

memory = MemorySaver()
graph = graph_builder.compile(checkpointer=memory)

config = {"configurable": {"thread_id": "1"}}

# 1. Run until the graph hits interrupt()
for event in graph.stream(
    {"messages": [{"role": "user", "content": "I need expert guidance on building an AI agent. Can you request assistance for me?"}]},
    config,
    stream_mode="values",
):
    if "messages" in event:
        event["messages"][-1].pretty_print()

# 2. Resume with the human's answer
human_command = Command(resume={"data": "We recommend LangGraph. It is more reliable and extensible than simple autonomous agents."})

for event in graph.stream(human_command, config, stream_mode="values"):
    if "messages" in event:
        event["messages"][-1].pretty_print()
```

---

## Project Structure

```
.
├── 1_basicchatbot.ipynb
├── Human_in_the_loop.ipynb
├── .env                # API keys (not committed)
├── .gitignore
└── README.md
```

---

## Key Concepts

- **State + reducers:** `Annotated[list, add_messages]` appends messages instead of overwriting them.
- **`tools_condition`:** prebuilt router that sends the graph to `tools` if the last message contains a tool call, otherwise to `END`.
- **Checkpointers:** `MemorySaver` stores state per `thread_id`, enabling multi-turn memory and resumable runs.
- **Stream modes:** `values` returns the full state after each step; `updates` returns only what each node changed.
- **`interrupt()` / `Command(resume=...)`:** pause execution for human input and continue from the same point.

---

## Notes

- Notebooks use `_____` on Groq. Swap the model name in `init_chat_model` / `ChatGroq` to use another provider or model.
- Tavily and Groq calls use your API quota. Keep keys out of version control.

---

## Resources

- [LangGraph docs](https://langchain-ai.github.io/langgraph/)
- [LangGraph human-in-the-loop guide](https://langchain-ai.github.io/langgraph/concepts/human_in_the_loop/)
- [Groq docs](https://console.groq.com/docs)
- [Tavily docs](https://docs.tavily.com/)

---

## License
