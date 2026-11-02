# I used an AI assistant for parts of the implementation but the design decisions, debugging and live testing are mine and I’m happy to walk through any of it


# GenAI Report Agent

An autonomous agent that **collects** recent articles from trusted sources (BBC News, GOV.UK), **summarises** them every hour into short, evaluated reports, and lets users **chat** with the latest reports — including vague questions like *“What’s happening nowadays?”*.

Built with **LangGraph** (both workflows are graphs; data collection runs through LangChain tools via `ToolNode`, generation through LangChain chat-model interfaces — no direct vendor API calls), **APScheduler** for the hourly run, **BeautifulSoup** for scraping, and **Streamlit** / CLI for chat.

```
           ┌─────────────── hourly (APScheduler) ───────────────┐
           ▼                                                    │
 plan_collection ─► collect (ToolNode: BBC RSS / GOV.UK Atom → scrape → clean)
                       ─► process (merge · dedupe · relevance rank · backfill)
                       ─► summarise (map-reduce, structured output)
                       ─► evaluate ──fail──► revise ─┐   (max 2 revisions)
                            │  ▲─────────────────────┘
                            └─pass─► save (JSON + Markdown)  ──►  data/reports/
                                                                      │
 chat:  analyse (intent · follow-up rewrite) ─► retrieve (latest-per-topic | BM25+recency) ─► respond (grounded, cited)
```

Full Mermaid diagrams of both graphs: [`docs/graphs.md`](docs/graphs.md). Design rationale: [`WRITEUP.md`](WRITEUP.md).

---

## Quick start

```bash
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env                                   # add ANTHROPIC_API_KEY or OPENAI_API_KEY
```

### 1. Try it in 10 seconds — no API key, no internet

```bash
python main.py demo
```

Runs three simulated hourly ticks on bundled **sample articles**, then a scripted chat and the evaluation suite. Without a key, the system uses its **offline baseline** (extractive summariser + retrieval-only answers) so everything is testable locally.

### 2. Real run (live sources + LLM)

```bash
python main.py report                         # one cycle now, all topics (manual trigger)
python main.py report --topic "AI Regulation" # single topic
python main.py schedule                       # run now, then every 60 min (SCHEDULE_MINUTES / --minutes)
python main.py chat                           # terminal chat over the latest reports
streamlit run streamlit_app.py                # web chat + report browser + "generate now" + background scheduler
python main.py eval                           # report + chat evaluations → examples/eval_results.md
```

Flags: `--offline` (sample articles instead of scraping) · `--no-llm` (force the extractive baseline) · `--data-dir PATH`.

### 3. Tests

```bash
pytest -q
```

10 tests cover feed parsing, HTML extraction/cleaning, topic filtering, the evaluator (catches invented numbers), cross-run de-duplication, the **LLM path including the evaluate → revise loop** (using a scripted fake chat model, so no key is needed), and chat behaviour (vague, follow-up, out-of-scope).

---

## Choosing an LLM

| `LLM_PROVIDER` | Needs | Default model (`LLM_MODEL` to override) |
|---|---|---|
| `auto` (default) | first of `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `OLLAMA_BASE_URL` | — |
| `anthropic` | `ANTHROPIC_API_KEY` | `claude-sonnet-5-5` |
| `openai` | `OPENAI_API_KEY` | `gpt-4o-mini` |
| `ollama` | local Ollama server | `llama3.1` |
| `offline` | nothing | extractive baseline |

All other settings live in `.env.example` (topics, schedule, article caps, revision budget, LLM-judge on/off).

## Report format

Each report (`data/reports/<timestamp>_<topic>_<id>.md` + `.json`) contains a **100–150 word summary**, **3–5 key takeaways**, **organisations / terms / topics**, numbered sources, and the **evaluation scorecard**. See [`examples/reports/`](examples/reports/).

## Logging & evaluation

* `logs/agent.jsonl` — structured JSON lines: every graph node’s duration, articles fetched/kept, eval checks, retrieval confidence, chat intent and citations (sample: [`examples/sample_log.jsonl`](examples/sample_log.jsonl)). Console shows the same events in readable form.
* **Per-report evaluation** (inside the graph): word window, takeaway count & distinctness, **numbers grounded in sources**, **entity grounding**, lexical support, article coverage, plus an optional **LLM-as-judge** faithfulness/relevance score. Failures feed concrete feedback to the `revise` node.
* **Chat evaluation** (`python main.py eval`): scripted single- and multi-turn cases checking intent routing, content, citations, follow-up topic resolution, refusal on out-of-scope questions, and latency → [`examples/eval_results.md`](examples/eval_results.md).

## Project layout

```
main.py                    CLI: report | schedule | chat | eval | demo
streamlit_app.py           web UI
report_agent/
  config.py                settings, topics → keywords, trusted sources
  sources.py               RSS/Atom discovery, BeautifulSoup extraction, cleaning, relevance
  tools.py                 LangChain @tools executed by LangGraph ToolNode
  llm.py                   provider-agnostic chat-model factory (+ offline mode)
  summarise.py             map-reduce LLM summarisation + extractive baseline + entity extraction
  evaluation.py            deterministic checks + LLM-as-judge
  report_graph.py          LangGraph reporting workflow + run_report_cycle()
  scheduler.py             APScheduler (blocking for CLI, background for Streamlit)
  retrieval.py             BM25 + recency over report chunks
  chat_graph.py            LangGraph chat agent with memory (MemorySaver)
  evals.py                 evaluation harness
  storage.py               JSON/Markdown persistence, seen-URL index, article cache
fixtures/                  FICTIONAL sample articles (offline demo/tests only)
tests/                     pytest suite + scripted fake LLM
examples/                  demo transcript, reports, eval results, sample logs
docs/graphs.md             Mermaid diagrams of both graphs
```

## Notes

* **About the example outputs.** The files in `examples/` were produced with `python main.py demo`, i.e. the **offline baseline on the fictional sample articles** (organisations such as “Office for AI Standards” or “Monetary Policy Board” are invented). Run `python main.py report` with an API key and internet access to produce real reports from BBC News / GOV.UK.
* Scraping is polite: feeds for discovery, a descriptive User-Agent, timeouts, at most `MAX_ARTICLES_PER_TOPIC` article pages per source per hour, and failed pages fall back to the feed summary. Site HTML changes over time; selectors live in `sources.extract_article_text`.
