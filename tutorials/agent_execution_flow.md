# Biomni Agent Execution Flow (kyle_demo.py)

This note captures the end-to-end call order when running `tutorials/kyle_demo.py`
with `A1(...)` followed by `agent.go(...)`. Links jump to code locations.

## 0) Entry / Import
- `tutorials/kyle_demo.py`: script entrypoint; constructs `A1` then calls `go`.
  - Link: `tutorials/kyle_demo.py`
- `from biomni.agent import A1` loads the agent implementation.
  - Link: `biomni/agent/__init__.py`
  - Link: `biomni/agent/a1.py`
- `a1.py` module top-level loads `.env` (so env vars exist before config init).
  - Link: `biomni/agent/a1.py#L18`
- `default_config = BiomniConfig()` is created at import time and reads env overrides.
  - Link: `biomni/config.py#L12`
  - Link: `biomni/config.py#L55`
  - Link: `biomni/config.py#L98`

## 1) Agent Construction (tools, LLM, know-how, workflow)
- `A1.__init__` fills missing params from `default_config`, prints configuration.
  - Link: `biomni/agent/a1.py#L56`
- `get_llm(...)` creates the actual LLM client based on model/source.
  - Link: `biomni/llm.py#L13`
- If `use_tool_retriever=True`, initialize tool registry + retriever.
  - Link: `biomni/agent/a1.py#L208`
- `KnowHowLoader()` loads know-how docs.
  - Link: `biomni/agent/a1.py#L212`
- `self.configure()` builds system prompt + LangGraph workflow (generate/execute).
  - Link: `biomni/agent/a1.py#L1285`

## 2) Collect Resources (candidates for LLM selection)
- `agent.go(prompt)` begins execution.
  - Link: `biomni/agent/a1.py#L1755`
- `_prepare_resources_for_retrieval(prompt)` collects tools/data/libraries/know-how.
  - Link: `biomni/agent/a1.py#L1642`

## 3) LLM Selects Resources (prompt-based retrieval)
- `ToolRetriever.prompt_based_retrieval(...)` builds a retrieval prompt.
  - Link: `biomni/model/retriever.py#L14`
- `_format_resources_for_prompt(...)` turns resources into numbered lists.
  - Link: `biomni/model/retriever.py#L134`
- LLM call for retrieval: `llm.invoke([HumanMessage(content=prompt)])`.
  - Link: `biomni/model/retriever.py#L98`
- `_parse_llm_response(...)` extracts selected indices.
  - Link: `biomni/model/retriever.py#L154`
- Slice by indices to get `selected_resources`.
  - Link: `biomni/model/retriever.py#L108`

## 4) Inject Selected Resources into System Prompt
- `update_system_prompt_with_selected_resources(...)` rebuilds the system prompt
  with selected tools/data/libraries/know-how.
  - Link: `biomni/agent/a1.py#L1825`

## 5) Main Loop: Generate ↔ Execute (LangGraph)
- `self.app.stream(...)` runs the LangGraph workflow.
  - Link: `biomni/agent/a1.py#L1776`
- `generate(state)` calls LLM with system prompt + messages.
  - Link: `biomni/agent/a1.py#L1377`
- If `<execute>` is present, route to `execute(state)`.
  - Link: `biomni/agent/a1.py#L1467`
- Execution routing:
  - `#!R` → `run_r_code` in `biomni/utils.py`.
    - Link: `biomni/utils.py#L25`
  - `#!BASH` / `#!CLI` → `run_bash_script` in `biomni/utils.py`.
    - Link: `biomni/utils.py#L56`
  - Otherwise → `run_python_repl` in `biomni/tool/support_tools.py`.
    - Link: `biomni/tool/support_tools.py#L13`
- Execution results are wrapped in `<observation>` and appended to messages.
  - Link: `biomni/agent/a1.py#L1546`
- Workflow loops `execute → generate` until `<solution>`; then `END`.
  - Link: `biomni/agent/a1.py#L1633`
