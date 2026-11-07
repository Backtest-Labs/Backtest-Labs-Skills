# Backtest Labs skills

Skills that teach an AI assistant to use the Backtest Labs MCP server well. Backtest Labs builds and backtests trading strategies on stocks and crypto. A strategy is a graph of indicator, operator and order nodes saved as a JSON document. A backtest runs each ticker on its own, with the costs, slippage and position sizing you set, and reports returns, drawdowns, the trade log, charts and ten validations. A completed backtest can be exported as Pine Script or Python signal code.

The MCP server (https://mcp.backtestlabs.com/mcp) exposes eighteen tools: account limits, ticker search, the strategy document reference, strategy create, read and update, the Strategy Hub, backtest start, status and cancel, results, trades, charts, code export and the documentation. The tools already describe themselves. These skills add what a tool list cannot: the order of calls, the defaults to choose when the user does not say, how to recover from a refused save, how to read a trade log or a validation without guessing, and when not to spend a backtest.

## Skills

| Skill | What it does |
|---|---|
| `backtestlabs-backtesting-a-strategy` | A described strategy to a saved document to a completed run to a first summary. Spec card, graph patterns, save-refusal recovery, the Strategy Hub as a starting point. |
| `backtestlabs-reading-backtest-results` | Review a completed run in a fixed order with a validity gate first, or explain a metric, validation or chart. Routes node and engine questions to the documentation tool. |
| `backtestlabs-trade-log-forensics` | Reads the trade log and the drawdown table to locate where an edge leaks: entries, exits, costs or regime. |
| `backtestlabs-exit-engineering` | Sets stop, target and trailing exits from the trades' own adverse and favorable excursions, builds the exit nodes, re-runs and compares. |
| `backtestlabs-robustness-audit` | Turns on every validation the plan includes, runs once, reads the ten results in a dependency order into a scorecard with no verdict. |
| `backtestlabs-walk-forward-parameter-selection` | Sweeps parameters with Walk Forward and Parameter Sensitivity, picks the plateau rather than the peak, updates the strategy once and re-runs with hold-out. |
| `backtestlabs-porting-and-exporting-code` | Ports a Pine Script or Python strategy into a node graph and back, and exports a completed backtest as Pine Script or Python with a parity check. |

## Data the skills send

Every tool call goes to the Backtest Labs MCP server over the user's own OAuth session. The skills send nothing anywhere else: no third-party service, no telemetry. What the server receives is the strategy documents the user asks to build, the run ids of the user's own backtests and the tool arguments shown in each skill.

## Using the skills

Install the plugin from the directory of your assistant, connect Backtest Labs when asked, and ask in plain language: "backtest a 20/50 EMA crossover on QQQ 4-hour bars", "why did my strategy lose money in 2022", "export my latest backtest as Pine Script". The matching skill loads on its own. Every backtest a skill starts counts against the plan's daily backtest runtime; the skills check the remaining time before starting one.

## Layout

```
.claude-plugin/plugin.json   manifest for the Claude plugin directory
.mcp.json                    the MCP server, for Claude
plugin.json                  manifest for the ChatGPT and Codex plugin package
mcp.json                     the MCP server, for ChatGPT and Codex
skills/<name>/SKILL.md       one folder per skill, with references/ beside it
```
