# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

TradingAgents is a multi-agent LLM financial trading framework (v0.2.2) that mirrors real trading firm structure. It uses LangGraph to orchestrate specialized agents through a pipeline: data gathering → researcher debate → trader decision → risk management debate → portfolio manager approval.

## Commands

```bash
# Install dependencies (uses UV lock file)
pip install .

# Run the CLI
python -m cli.main
tradingagents               # if installed via pip

# Run a quick example
python main.py

# Run tests
python -m pytest tests/
python -m pytest tests/test_ticker_symbol_handling.py  # single test file

# Run a specific test
python -m pytest tests/test_ticker_symbol_handling.py::TestNormalizeTickerSymbol::test_basic_ticker
```

## Environment Setup

Copy `.env.example` to `.env` and set API keys for your chosen LLM provider (OpenAI, Anthropic, Google, xAI, OpenRouter, or Ollama).

## Architecture

### Agent Pipeline (LangGraph)

The graph in `tradingagents/graph/` orchestrates this flow:

1. **Analyst Team** (`tradingagents/agents/analysts/`) — 4 agents run in parallel (configurable): fundamentals, market/technical, news, social media. Each produces a report stored in `AgentState`.
2. **Researcher Debate** (`tradingagents/agents/researchers/`) — Bull and Bear researchers debate investment merits. Rounds are configurable. Research Manager synthesizes.
3. **Trader** (`tradingagents/agents/trader/`) — Generates investment plan from researcher synthesis.
4. **Risk Debate** (`tradingagents/agents/risk_mgmt/`) — Aggressive, Conservative, and Neutral debaters evaluate risk.
5. **Portfolio Manager** (`tradingagents/agents/managers/portfolio_manager.py`) — Final approval/rejection with reasoning.

### Key Files

| File | Purpose |
|------|---------|
| `tradingagents/graph/trading_graph.py` | Main `TradingAgentsGraph` class — primary API entry point |
| `tradingagents/graph/setup.py` | LangGraph node/edge wiring |
| `tradingagents/graph/conditional_logic.py` | Routing between debate rounds and next steps |
| `tradingagents/agents/utils/agent_states.py` | `AgentState`, `InvestDebateState`, `RiskDebateState` TypedDicts |
| `tradingagents/agents/utils/memory.py` | `FinancialSituationMemory` — per-role persistent memory |
| `tradingagents/dataflows/interface.py` | Data vendor abstraction layer |
| `tradingagents/llm_clients/factory.py` | Multi-provider LLM factory |
| `tradingagents/default_config.py` | Default configuration values |

### Data Layer

`tradingagents/dataflows/` supports two data vendors:
- **yFinance** — free, used by default
- **Alpha Vantage** — premium, separate modules per data type (stock, fundamentals, indicators, news)

The interface in `dataflows/interface.py` abstracts both vendors; swap via config.

### LLM Configuration

Two LLMs are used per run:
- `deep_think_llm` — complex reasoning (analyst reports, debates) — defaults to `gpt-5.2`
- `quick_think_llm` — fast responses (routing, formatting) — defaults to `gpt-5-mini`

Configure via `TradingAgentsGraph(config={...})` or `tradingagents/default_config.py`.

### Python API Usage

```python
from tradingagents.graph.trading_graph import TradingAgentsGraph

ta = TradingAgentsGraph(debug=True, config={
    "llm_provider": "anthropic",
    "deep_think_llm": "claude-opus-4-6",
    "quick_think_llm": "claude-haiku-4-5-20251001",
    "online_tools": True,
})
state, decision = ta.propagate("NVDA", "2024-05-10")
```

### State Flow

State TypedDicts in `agent_states.py` carry data through the graph. Key fields:
- `analyst_reports` dict — populated by each analyst
- `investment_debate_state` — bull/bear debate messages
- `risk_debate_state` — risk team debate messages
- `final_trade_decision` — portfolio manager's verdict
