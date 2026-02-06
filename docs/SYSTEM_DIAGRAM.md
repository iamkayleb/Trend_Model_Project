# System Diagram - Trend Model Project

> Volatility-adjusted trend portfolio construction and backtesting platform.

---

## High-Level Architecture

```
 ┌──────────────────────────────────────────────────────────────────────────────┐
 │                          USER INTERFACES                                     │
 │                                                                              │
 │  ┌──────────┐  ┌─────────────────┐  ┌──────────────┐  ┌──────────────────┐  │
 │  │ CLI      │  │ Streamlit Web   │  │ Jupyter GUI  │  │ FastAPI Server   │  │
 │  │ trend *  │  │ streamlit_app/  │  │ gui/app.py   │  │ api_server/      │  │
 │  └────┬─────┘  └───────┬─────────┘  └──────┬───────┘  └────────┬─────────┘  │
 └───────┼────────────────┼────────────────────┼───────────────────┼────────────┘
         │                │                    │                   │
         └────────────────┴──────────┬─────────┴───────────────────┘
                                     │
                                     ▼
 ┌──────────────────────────────────────────────────────────────────────────────┐
 │                       CORE API LAYER                                         │
 │                                                                              │
 │           trend_analysis/api.py  →  run_simulation()                         │
 │                          │                                                   │
 │           trend_analysis/pipeline.py  →  orchestration                       │
 │           trend_analysis/pipeline_runner.py  →  execution + diagnostics      │
 │           trend_analysis/pipeline_entrypoints.py  →  config binding          │
 └───────────────────────────────┬──────────────────────────────────────────────┘
                                 │
            ┌────────────────────┼────────────────────┐
            ▼                    ▼                    ▼
 ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
 │ PREPROCESSING    │ │ SELECTION        │ │ PORTFOLIO        │
 │ stages/          │ │ stages/          │ │ stages/          │
 │ preprocessing.py │ │ selection.py     │ │ portfolio.py     │
 │                  │ │                  │ │                  │
 │ - Freq detection │ │ - Rank funds     │ │ - Signal compute │
 │ - Calendar align │ │ - Risk-free ID   │ │ - Weight calc    │
 │ - Missing policy │ │ - Universe mask  │ │ - Regime overlay │
 │ - Window build   │ │ - Metric bundle  │ │ - Return calc    │
 └──────────────────┘ └──────────────────┘ └──────────────────┘
```

---

## Entry Points & Interfaces

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              ENTRY POINTS                                   │
├──────────────────┬──────────────────────────────┬──────────────────────────┤
│ Entry Point      │ Source File                   │ Invocation               │
├──────────────────┼──────────────────────────────┼──────────────────────────┤
│ CLI              │ src/trend/cli.py              │ trend run|app|nl|report  │
│ Streamlit App    │ streamlit_app/app.py          │ streamlit run app.py     │
│ Python API       │ src/trend_analysis/api.py     │ run_simulation(cfg, df)  │
│ Jupyter GUI      │ src/trend_analysis/gui/app.py │ gui.launch()             │
│ FastAPI Server   │ src/trend_analysis/api_server │ uvicorn ...              │
│ LLM Proxy        │ src/trend_analysis/llm_proxy  │ trend proxy              │
└──────────────────┴──────────────────────────────┴──────────────────────────┘

CLI Subcommands (src/trend/cli.py):
 trend
  ├── run        Run analysis pipeline with config + data
  ├── report     Generate unified report from results
  ├── stress     Run stress-test scenarios
  ├── app        Launch Streamlit web UI
  ├── nl         Natural language config editing (LLM-powered)
  ├── explain    LLM-powered result explanation
  ├── check      Environment validation
  ├── schema     Dump JSON schema for config
  ├── proxy      Start LLM proxy service
  └── coverage   Config coverage tracking
```

---

## Core Analysis Pipeline

The pipeline executes in three sequential stages, orchestrated by `pipeline_runner.py`:

```
                    ┌─────────────────────────┐
                    │  Configuration (YAML)    │
                    │  config/defaults.yml     │
                    │  config/presets/*.yml    │
                    └────────────┬────────────┘
                                 │  load + validate
                                 ▼
                    ┌─────────────────────────┐
                    │  Config Validation       │
                    │  config/models.py        │ Pydantic models
                    │  config/validation.py    │ Business rules
                    │  config/schema_valid...  │ JSON schema
                    └────────────┬────────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
         ▼                       ▼                       ▼
  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐
  │  Data Input   │    │  Signal Config   │    │  Weight Policy   │
  │  CSV / Excel  │    │  Trend windows   │    │  risk_parity     │
  │  market_data  │    │  Vol-adjust      │    │  hrp / erc       │
  │  io/          │    │  Lookback        │    │  robust          │
  └───────┬───────┘    └────────┬─────────┘    └────────┬─────────┘
          │                     │                       │
          └─────────────────────┼───────────────────────┘
                                │
                                ▼
 ══════════════════════════════════════════════════════════════════
 ║                   PIPELINE EXECUTION                          ║
 ══════════════════════════════════════════════════════════════════

  Stage 1: PREPROCESSING (stages/preprocessing.py)
  ┌──────────────────────────────────────────────────────────────┐
  │  Input: Raw price/return DataFrame                           │
  │                                                              │
  │  1. Detect data frequency (daily / weekly / monthly)         │
  │  2. Apply calendar alignment (normalize to month-end)        │
  │  3. Handle missing data (fill / drop per policy)             │
  │  4. Build sample windows (in-sample / out-of-sample)         │
  │                                                              │
  │  Output: _PreprocessStage, _WindowStage                      │
  └──────────────────────────┬───────────────────────────────────┘
                             │
                             ▼
  Stage 2: SELECTION (stages/selection.py)
  ┌──────────────────────────────────────────────────────────────┐
  │  Input: Preprocessed data + config                           │
  │                                                              │
  │  1. Identify risk-free fund from data columns                │
  │  2. Apply universe membership masks (time-varying)           │
  │  3. Compute metric bundle (return, vol, Sharpe per fund)     │
  │  4. Rank and select top-N funds by configured criteria       │
  │                                                              │
  │  Output: _SelectionStage (fund_cols, risk_free, metrics)     │
  └──────────────────────────┬───────────────────────────────────┘
                             │
                             ▼
  Stage 3: PORTFOLIO (stages/portfolio.py)
  ┌──────────────────────────────────────────────────────────────┐
  │  Input: Selected funds + config + windows                    │
  │                                                              │
  │  1. Compute trend signals (signals.py → TrendSpec)           │
  │  2. Derive positions from signals                            │
  │  3. Calculate constrained weights (risk.py)                  │
  │  4. Apply weight policy (risk parity / HRP / ERC / robust)  │
  │  5. Apply regime overlay if configured (regimes.py)          │
  │  6. Calculate portfolio returns and turnover                 │
  │  7. Compute performance statistics (metrics/*.py)            │
  │     - Sharpe, Sortino, Calmar, Max Drawdown, IR              │
  │  8. Assemble analysis output with diagnostics                │
  │                                                              │
  │  Output: PipelineResult → RunResult                          │
  └──────────────────────────┬───────────────────────────────────┘
                             │
                             ▼
                    ┌─────────────────────────┐
                    │  EXPORT & REPORTING      │
                    │  export/bundle.py        │
                    │  reporting/narrative.py   │
                    │                          │
                    │  Formats:                │
                    │  - Excel (.xlsx)         │
                    │  - CSV                   │
                    │  - JSON                  │
                    │  - PDF (fpdf2)           │
                    │  - HTML                  │
                    │  - ZIP bundle            │
                    └─────────────────────────┘
```

---

## Module Dependency Map

```
src/
├── trend/                          ◄── Unified CLI & utilities
│   ├── cli.py                      Main CLI: imports from trend_analysis.api,
│   │                               trend_analysis.config, trend_analysis.llm,
│   │                               trend_analysis.export, trend.reporting
│   ├── config_schema.py            Lightweight config loader (startup)
│   ├── validation.py               Frame validation, execution lag checks
│   ├── diagnostics.py              DiagnosticPayload, DiagnosticResult
│   └── reporting/                  Report generation (quick_summary, unified)
│
├── trend_analysis/                 ◄── Core engine (all analysis logic)
│   ├── api.py                      run_simulation() → pipeline orchestration
│   │   └── imports: pipeline.py, diagnostics.py, util/, weights/
│   │
│   ├── pipeline.py                 Re-exports from pipeline_helpers, pipeline_runner,
│   │   └── imports:                pipeline_entrypoints, stages/*, core/, data,
│   │                               metrics, perf, portfolio, regimes, risk, signals
│   │
│   ├── pipeline_helpers.py         Config parsing, signal/position computation
│   ├── pipeline_runner.py          _run_analysis_with_diagnostics()
│   ├── pipeline_entrypoints.py     ConfigBindings, run_from_config()
│   │
│   ├── config/                     ◄── Configuration subsystem
│   │   ├── models.py               Pydantic config models (ConfigProtocol)
│   │   ├── validation.py           Business rule validation
│   │   ├── schema_validation.py    JSON schema validation
│   │   ├── schema_generator.py     Schema generation from models
│   │   ├── bridge.py               Config format bridging
│   │   ├── patch.py                Config patching (for NL editing)
│   │   └── coverage.py             Config key usage tracking
│   │
│   ├── stages/                     ◄── Pipeline stages
│   │   ├── preprocessing.py        Data cleaning + window construction
│   │   ├── selection.py            Fund ranking + universe filtering
│   │   └── portfolio.py            Signal → weights → returns → stats
│   │
│   ├── metrics/                    ◄── Financial metrics
│   │   ├── summary.py              Period-level summaries
│   │   ├── rolling.py              Rolling window metrics
│   │   ├── attribution.py          Return attribution
│   │   └── turnover.py             Turnover calculation
│   │
│   ├── weights/                    ◄── Weighting strategies
│   │   ├── risk_parity.py          Inverse-vol risk parity
│   │   ├── hierarchical_risk_parity.py  HRP (clustering-based)
│   │   ├── equal_risk_contribution.py   ERC optimization
│   │   ├── robust_weighting.py     Bayesian shrinkage weighting
│   │   └── robust_config.py        Robustness configuration
│   │
│   ├── llm/                        ◄── LLM integration (optional)
│   │   ├── chain.py                ConfigPatchChain, ResultSummaryChain
│   │   ├── providers.py            OpenAI / Anthropic / Ollama adapters
│   │   ├── prompts.py              Prompt templates
│   │   ├── schema.py               LLM-friendly schema formatting
│   │   ├── validation.py           LLM output validation
│   │   ├── tracing.py              LangSmith tracing
│   │   ├── result_validation.py    Claim verification
│   │   ├── nl_logging.py           NL operation audit log
│   │   └── replay.py               Deterministic replay
│   │
│   ├── io/                         ◄── Data I/O
│   │   ├── market_data.py          Market data loading
│   │   ├── ui_ingest.py            UI upload ingestion
│   │   ├── validators.py           Data validators
│   │   └── date_correction.py      Date format fixing
│   │
│   ├── export/                     ◄── Output export
│   │   └── bundle.py               Multi-format export engine
│   │
│   ├── reporting/                  ◄── Report generation
│   │   ├── narrative.py            Narrative text generation
│   │   ├── portfolio_series.py     Portfolio series selection
│   │   └── run_artifacts.py        Run artifact packaging
│   │
│   ├── core/                       ◄── Core algorithms
│   │   ├── rank_selection.py       Fund ranking engine
│   │   └── metric_cache.py         Metric computation cache
│   │
│   ├── backtesting/                ◄── Backtesting
│   │   ├── bootstrap.py            Bootstrap simulation
│   │   └── harness.py              Walk-forward harness
│   │
│   ├── multi_period/               ◄── Multi-period analysis
│   │   ├── engine.py               Multi-period execution
│   │   ├── loaders.py              Period data loading
│   │   └── scheduler.py            Period scheduling
│   │
│   ├── perf/                       ◄── Performance optimization
│   │   ├── cache.py                General caching
│   │   ├── rolling_cache.py        Rolling computation cache
│   │   └── timing.py               Performance timing
│   │
│   ├── signals.py                  TrendSpec, compute_trend_signals()
│   ├── risk.py                     Constrained weight optimization
│   ├── portfolio.py                apply_weight_policy()
│   ├── regimes.py                  Market regime detection
│   ├── data.py                     Risk-free fund identification, CSV loading
│   ├── diagnostics.py              PipelineResult, reason codes
│   ├── constants.py                Shared constants
│   ├── presets.py                  Strategy presets
│   └── signal_presets.py           Signal parameter presets
│
├── analysis/                       ◄── Results analytics
│   └── results.py                  Results container, metadata builder
│
├── data/                           ◄── Data contracts
│   └── contracts.py                Input/output data contracts
│
├── backtest/                       ◄── Additional backtesting utilities
├── health_summarize/               ◄── Health check summaries
└── utils/                          ◄── Shared utilities
    └── path_resolution.py          Path resolvers
```

---

## Streamlit Web Application

```
streamlit_app/
│
├── app.py ─────────────────── Main entry: Demo + Custom Analysis modes
│   │
│   ├── components/demo_runner.py ── Demo preset execution
│   │   └── calls: trend_analysis.api.run_simulation()
│   │
│   └── state.py ──────────────── Session state management (central store)
│
├── pages/
│   ├── 1_Data.py ─────────── Data upload & validation
│   │   └── uses: components/upload_guard.py
│   │         components/csv_validation.py
│   │         components/data_schema.py
│   │         components/date_correction.py
│   │
│   ├── 2_Model.py ────────── Configuration editing UI
│   │   └── uses: components/llm_settings.py
│   │         state.py (config bridge)
│   │         config_bridge.py
│   │
│   ├── 3_Results.py ──────── Results visualization
│   │   └── uses: components/charts.py
│   │         components/explain_results.py
│   │         components/comparison.py
│   │
│   ├── 4_Help.py ─────────── Documentation display
│   │
│   └── 8_Validation.py ───── Data validation checks
│       └── uses: components/guardrails.py
│
└── components/
    ├── analysis_runner.py ── Run button → trend_analysis.api
    ├── demo_runner.py ────── Preset loader → run_simulation()
    ├── charts.py ─────────── Matplotlib/Streamlit charts
    ├── data_schema.py ────── Schema validation display
    ├── csv_validation.py ─── CSV format checks
    ├── date_correction.py ── Date fixing UI
    ├── upload_guard.py ────── File upload safety
    ├── llm_settings.py ───── LLM provider config UI
    ├── progress_eta.py ───── Progress bar + ETA
    ├── explain_results.py ── LLM-powered explanations
    ├── comparison.py ─────── Portfolio comparison
    ├── guardrails.py ─────── Safety checks
    └── data_cache.py ─────── Session data caching
```

---

## Configuration System

```
                         ┌──────────────────┐
                         │  User Input       │
                         │  (YAML file or    │
                         │   UI parameters   │
                         │   or NL command)  │
                         └────────┬─────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    ▼             ▼              ▼
             ┌───────────┐ ┌──────────┐  ┌────────────┐
             │ YAML Load │ │ UI State │  │ NL → LLM   │
             │ load_config│ │ Bridge   │  │ chain.py   │
             └─────┬─────┘ └────┬─────┘  └──────┬─────┘
                   │            │               │
                   └────────────┼───────────────┘
                                ▼
                   ┌─────────────────────────┐
                   │  Config Validation       │
                   │                          │
                   │  1. Schema validation    │ ← config.schema.json (77K lines)
                   │     (JSON Schema)        │
                   │                          │
                   │  2. Pydantic models      │ ← config/models.py
                   │     (type + constraint)  │
                   │                          │
                   │  3. Business rules       │ ← config/validation.py
                   │     (cross-field logic)  │
                   └────────────┬────────────┘
                                │
                  ┌─────────────┼──────────────┐
                  ▼             ▼              ▼
           ┌───────────┐ ┌───────────┐ ┌────────────────┐
           │ defaults   │ │ presets   │ │ user overrides │
           │ .yml       │ │ .yml x4  │ │                │
           └─────┬─────┘ └─────┬─────┘ └──────┬─────────┘
                 │             │              │
                 └─────────────┼──────────────┘
                               ▼
                  ┌─────────────────────────┐
                  │  Merged Configuration    │
                  │  (defaults ← preset     │
                  │   ← user overrides)     │
                  └─────────────────────────┘

Preset Strategies (config/presets/):
  ├── conservative.yml   Conservative allocation
  ├── balanced.yml       Balanced risk/return
  ├── aggressive.yml     Higher concentration
  └── cash_constrained.yml  Cash allocation limits
```

---

## LLM Integration (Optional)

```
                    ┌───────────────────────────┐
                    │  User Request              │
                    │  "Make it more aggressive" │
                    │  or "Explain results"      │
                    └─────────────┬─────────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    ▼                            ▼
           ┌────────────────┐          ┌────────────────┐
           │ NL Config Edit │          │ Result Explain  │
           │ ConfigPatchChain│         │ ResultSummary   │
           │                │          │ Chain           │
           └───────┬────────┘          └───────┬────────┘
                   │                           │
                   ▼                           ▼
           ┌────────────────────────────────────────────┐
           │  LLM Provider Layer (llm/providers.py)     │
           │                                            │
           │  ┌──────────┐ ┌──────────┐ ┌────────────┐ │
           │  │ OpenAI   │ │Anthropic │ │  Ollama    │ │
           │  │ GPT-4    │ │ Claude   │ │  (local)   │ │
           │  └──────────┘ └──────────┘ └────────────┘ │
           └───────────────────┬────────────────────────┘
                               │
                    ┌──────────┼──────────┐
                    ▼                     ▼
           ┌────────────────┐    ┌────────────────┐
           │ Config Patch   │    │ Narrative Text  │
           │ (validated     │    │ + Claim Issues  │
           │  YAML diff)    │    │ (fact-checked)  │
           └────────────────┘    └────────────────┘

Supporting modules:
  llm/prompts.py          Prompt templates for each chain
  llm/schema.py           Compact schema for LLM context windows
  llm/validation.py       Output validation + guardrails
  llm/tracing.py          LangSmith observability
  llm/result_validation.py  Claim verification against actual metrics
  llm/nl_logging.py       Audit log for NL operations
  llm/replay.py           Deterministic replay for testing
```

---

## Weighting Strategies

```
                    ┌──────────────────────────┐
                    │  Selected Fund Returns    │
                    │  + Covariance Matrix      │
                    └─────────────┬────────────┘
                                  │
                   ┌──────────────┼──────────────┐
                   ▼              ▼               ▼
          ┌──────────────┐ ┌────────────┐ ┌───────────────┐
          │ Risk Parity  │ │    HRP     │ │     ERC       │
          │ (inverse-vol)│ │(clustering)│ │(optimization) │
          │              │ │            │ │               │
          │ risk_parity  │ │hierarchical│ │ equal_risk_   │
          │ .py          │ │_risk_parity│ │ contribution  │
          │              │ │ .py        │ │ .py           │
          └──────┬───────┘ └─────┬──────┘ └───────┬───────┘
                 │               │                │
                 └───────────────┼────────────────┘
                                 │
                    ┌────────────┼────────────┐
                    ▼                         ▼
           ┌────────────────┐       ┌────────────────┐
           │ Robust Wrapper │       │ Direct Weights │
           │ Bayesian       │       │ (pass-through) │
           │ shrinkage      │       └────────────────┘
           │ robust_        │
           │ weighting.py   │
           └────────┬───────┘
                    │
                    ▼
           ┌────────────────────────────────┐
           │  Constrained Weights (risk.py) │
           │  - Position limits             │
           │  - Turnover caps               │
           │  - Cash allocation bounds      │
           │  - Volatility targeting        │
           └────────────────────────────────┘
```

---

## Data Flow

```
 ┌──────────────┐     ┌───────────────┐     ┌─────────────────┐
 │ CSV / Excel  │     │ Universe Mask │     │ Config YAML     │
 │ (returns or  │     │ (membership   │     │ (parameters)    │
 │  prices)     │     │  over time)   │     │                 │
 └──────┬───────┘     └───────┬───────┘     └────────┬────────┘
        │                     │                      │
        ▼                     ▼                      ▼
 ┌──────────────────────────────────────────────────────────────┐
 │                    DATA LOADING & VALIDATION                  │
 │                                                               │
 │  io/market_data.py    Load CSV/Excel, detect format           │
 │  io/validators.py     Column checks, date parsing             │
 │  io/date_correction   Fix ambiguous date formats              │
 │  io/ui_ingest.py      Streamlit upload processing             │
 │  data.py              Risk-free fund identification           │
 │  input_validation.py  Schema + business rule checks           │
 └──────────────────────────────┬────────────────────────────────┘
                                │
                                ▼
 ┌──────────────────────────────────────────────────────────────┐
 │                    ANALYSIS ENGINE                             │
 │                                                               │
 │  Preprocessing → Selection → Portfolio                        │
 │  (see "Core Analysis Pipeline" above)                         │
 └──────────────────────────────┬────────────────────────────────┘
                                │
                                ▼
 ┌──────────────────────────────────────────────────────────────┐
 │                    OUTPUT GENERATION                           │
 │                                                               │
 │  RunResult                                                    │
 │  ├── Portfolio returns (pd.Series)                            │
 │  ├── Fund weights over time (pd.DataFrame)                    │
 │  ├── Performance metrics (dict)                               │
 │  ├── Diagnostics (DiagnosticPayload)                          │
 │  └── Metadata (run config, timestamps, versions)              │
 │                                                               │
 │  Export Formats:                                               │
 │  ├── Excel    → summary sheet + per-period detail             │
 │  ├── CSV      → flat metric tables                            │
 │  ├── JSON     → machine-readable results                      │
 │  ├── PDF      → formatted report (fpdf2)                      │
 │  ├── HTML     → interactive report                            │
 │  └── ZIP      → reproducibility bundle (config + data + out)  │
 └──────────────────────────────────────────────────────────────┘
```

---

## Multi-Period Analysis

```
 ┌─────────────────────────────────────────────────┐
 │  Multi-Period Engine (multi_period/engine.py)    │
 │                                                  │
 │  Scheduler generates period windows:             │
 │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐   │
 │  │Period 1│ │Period 2│ │Period 3│ │Period N│   │
 │  │2015-17 │ │2016-18 │ │2017-19 │ │  ...   │   │
 │  └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘   │
 │      │          │          │          │         │
 │      ▼          ▼          ▼          ▼         │
 │  ┌──────────────────────────────────────────┐   │
 │  │  Each period → run_simulation()           │   │
 │  │  (independent pipeline execution)         │   │
 │  └──────────────────────────────────────────┘   │
 │      │          │          │          │         │
 │      ▼          ▼          ▼          ▼         │
 │  ┌──────────────────────────────────────────┐   │
 │  │  Aggregate: rolling metrics, stability,   │   │
 │  │  consistency scores, walk-forward stats   │   │
 │  └──────────────────────────────────────────┘   │
 └─────────────────────────────────────────────────┘

 Backtesting (backtesting/):
  ├── bootstrap.py   Statistical bootstrap for confidence intervals
  └── harness.py     Walk-forward backtesting harness
```

---

## Performance & Caching

```
 ┌──────────────────────────────────────────────────┐
 │  Performance Layer (perf/)                        │
 │                                                   │
 │  cache.py          General memoization cache      │
 │  rolling_cache.py  Rolling computation cache      │
 │                    (dataset hash → cached result)  │
 │  timing.py         Execution timing + profiling   │
 │                                                   │
 │  core/metric_cache.py  Per-window metric cache    │
 └──────────────────────────────────────────────────┘
```

---

## CI/CD & GitHub Workflows

```
 ┌──────────────────────────────────────────────────────────────────┐
 │                     GITHUB WORKFLOWS (.github/workflows/)        │
 │                                                                  │
 │  ┌──────────────────────────────────────────────────────────┐   │
 │  │  CI PIPELINE                                              │   │
 │  │                                                           │   │
 │  │  ci.yml ────────── Lint → Format → Type-check → Test      │   │
 │  │                    (Python 3.11 + 3.12, ruff, mypy)       │   │
 │  │                                                           │   │
 │  │  pr-00-gate.yml ── PR quality gate (smoke + coverage)     │   │
 │  │  pr-11-ci-smoke ── Quick smoke tests on PR                │   │
 │  │  pr-12-playwright  Browser tests for Streamlit UI         │   │
 │  │  autofix.yml ───── Auto-fix lint/format violations        │   │
 │  └──────────────────────────────────────────────────────────┘   │
 │                                                                  │
 │  ┌──────────────────────────────────────────────────────────┐   │
 │  │  AI AGENT SYSTEM (synced from stranske/Workflows)         │   │
 │  │                                                           │   │
 │  │  agents-orchestrator.yml ─────── 30-min sweep cycle       │   │
 │  │       │                                                   │   │
 │  │       ├── agents-issue-intake ── Issue → PR bootstrap     │   │
 │  │       ├── agents-keepalive-loop  Codex task execution     │   │
 │  │       ├── agents-verifier ────── Result verification      │   │
 │  │       └── agents-guard ──────── Safety checks             │   │
 │  │                                                           │   │
 │  │  agents-71-*-dispatcher ──────── Task dispatch            │   │
 │  │  agents-72-*-worker ─────────── Task execution            │   │
 │  │  agents-73-*-conveyor ───────── Task chaining             │   │
 │  │  agents-80-pr-event-hub ─────── PR event routing          │   │
 │  │  agents-81-gate-followups ───── Post-gate actions         │   │
 │  │  agents-auto-label ──────────── Issue/PR labeling         │   │
 │  │  agents-autofix-loop ────────── Auto-fix iterations       │   │
 │  │  agents-pr-meta ─────────────── PR metadata extraction    │   │
 │  └──────────────────────────────────────────────────────────┘   │
 │                                                                  │
 │  ┌──────────────────────────────────────────────────────────┐   │
 │  │  MAINTENANCE                                              │   │
 │  │                                                           │   │
 │  │  dependabot-automerge.yml ── Auto-merge dep updates       │   │
 │  │  maint-coverage-guard.yml ── Coverage enforcement         │   │
 │  │  settings-effectiveness ──── Settings validation          │   │
 │  └──────────────────────────────────────────────────────────┘   │
 └──────────────────────────────────────────────────────────────────┘

 Workflow Source:
  ┌────────────────────────┐       sync        ┌──────────────────┐
  │ stranske/Workflows     │ ──────────────── → │ This repo        │
  │ (central library)      │  agents-*.yml      │ .github/workflows│
  │                        │  autofix.yml       │                  │
  │ reusable-*.yml@v1      │  pr-00-gate.yml    │ ci.yml (local)   │
  └────────────────────────┘                    └──────────────────┘
```

---

## Test Suite Structure

```
 tests/
 ├── unit/                         Pure unit tests
 ├── smoke/                        Quick validation tests
 ├── golden/                       Golden file regression tests
 │
 ├── trend_analysis/               Core engine tests
 │   ├── test_config*.py           Configuration validation
 │   ├── test_metrics*.py          Financial metric accuracy
 │   ├── test_weights*.py          Weighting strategy tests
 │   ├── test_pipeline*.py         Pipeline integration
 │   ├── test_signals*.py          Signal computation
 │   ├── test_risk*.py             Risk computation
 │   ├── test_llm*.py              LLM integration tests
 │   ├── test_export*.py           Export format tests
 │   └── test_multi_period*.py     Multi-period tests
 │
 ├── app/                          Streamlit app tests
 ├── backtesting/                  Backtesting tests
 ├── scripts/                      Script tests
 ├── tools/                        Tool tests
 ├── proxy/                        Proxy tests
 │
 ├── workflows/                    GitHub workflow tests
 │   ├── github_scripts/           Action script tests
 │   └── fixtures/                 Workflow test fixtures
 │       ├── orchestrator/
 │       ├── keepalive/
 │       └── agents_pr_meta/
 │
 ├── soft_coverage/                Coverage quality tests
 ├── fixtures/                     Common test fixtures
 └── data/                         Test data files

 Testing stack: pytest + coverage + mypy + ruff + black + pre-commit
```

---

## Infrastructure & Deployment

```
 ┌─────────────────────────────────────────────────────────────┐
 │  DOCKER                                                     │
 │                                                             │
 │  Dockerfile (multi-stage)                                   │
 │  ┌─────────────┐     ┌──────────────┐                      │
 │  │ Builder     │     │ Runtime      │                       │
 │  │ Python 3.12 │ ──→ │ Python slim  │                       │
 │  │ + deps      │     │ + venv only  │                       │
 │  └─────────────┘     └──────────────┘                      │
 │                                                             │
 │  docker-compose.yml                                         │
 │  ├── app service (Streamlit on port 8501)                   │
 │  └── volumes for config + data                              │
 └─────────────────────────────────────────────────────────────┘

 ┌─────────────────────────────────────────────────────────────┐
 │  DEVELOPMENT ENVIRONMENT                                    │
 │                                                             │
 │  .devcontainer/devcontainer.json   VSCode dev container     │
 │  .pre-commit-config.yaml          Pre-commit hooks          │
 │  pyproject.toml                    Package config + deps     │
 │  pytest.ini                        Test configuration        │
 │  .flake8 / .hadolint.yaml         Linting rules             │
 └─────────────────────────────────────────────────────────────┘
```

---

## Key Dependencies

```
 Core                          Optional (app)              Optional (llm)
 ─────────────────             ─────────────────           ─────────────────
 pandas 2.3.3                  streamlit 1.53.1            langchain ~1.2-1.3
 numpy 2.4.1                   fastapi 0.128.0             langsmith 0.6.4
 scipy 1.17.0                  uvicorn 0.40.0              openai
 pydantic 2.12.5               httpx 0.28.1                anthropic
 PyYAML 6.0.3                                              ollama
 jsonschema 4.26.0             Dev
 pandera 0.28.1                ─────────────────
 joblib 1.5.3                  pytest 9.0.2
 openpyxl 3.1.5                mypy 1.19.1
 xlsxwriter 3.2.9              black 26.1.0
 fpdf2 2.8.5                   ruff 0.14.14
 matplotlib 3.10.8             pre-commit 4.5.1
 ipywidgets 8.1.8
```

---

## Cross-Cutting Concerns

```
 ┌──────────────────────────────────────────────────────────────────┐
 │  DIAGNOSTICS                                                     │
 │  trend/diagnostics.py + trend_analysis/diagnostics.py            │
 │  - PipelineResult wraps success/failure with reason codes         │
 │  - DiagnosticPayload captures full execution context              │
 │  - Reason codes enable programmatic error handling                │
 └──────────────────────────────────────────────────────────────────┘

 ┌──────────────────────────────────────────────────────────────────┐
 │  LOGGING                                                         │
 │  trend_analysis/logging.py + logging_setup.py                    │
 │  - Structured logging with step-level granularity                │
 │  - Configurable verbosity                                        │
 └──────────────────────────────────────────────────────────────────┘

 ┌──────────────────────────────────────────────────────────────────┐
 │  REPRODUCIBILITY                                                 │
 │  export/bundle.py → ZIP bundles                                  │
 │  - Config snapshot + data hash + output + metadata                │
 │  - Enables exact result reproduction                              │
 └──────────────────────────────────────────────────────────────────┘

 ┌──────────────────────────────────────────────────────────────────┐
 │  CONFIG COVERAGE                                                 │
 │  config/coverage.py                                              │
 │  - Tracks which config keys are actually read during a run        │
 │  - Identifies unused/dead configuration                           │
 └──────────────────────────────────────────────────────────────────┘
```

---

## Summary: How Everything Connects

```
 USER
  │
  ├─ CLI (trend run) ──────────────────┐
  ├─ Streamlit (web UI) ──────────────┤
  ├─ Jupyter (gui.launch()) ──────────┤
  ├─ FastAPI (REST) ──────────────────┤
  │                                    │
  │                                    ▼
  │                          ┌──────────────────┐
  │                          │ run_simulation()  │ ← Core API
  │                          └────────┬─────────┘
  │                                   │
  │                    ┌──────────────┼──────────────┐
  │                    ▼              ▼              ▼
  │              Preprocess      Selection     Portfolio
  │              (clean data)    (rank funds)  (build portfolio)
  │                    │              │              │
  │                    └──────────────┼──────────────┘
  │                                   │
  │                          ┌────────┴────────┐
  │                          ▼                 ▼
  │                    Single-Period      Multi-Period
  │                    Analysis           Walk-Forward
  │                          │                 │
  │                          └────────┬────────┘
  │                                   │
  │                                   ▼
  │                          ┌──────────────────┐
  │                          │  RunResult        │
  │                          │  (returns, wts,   │
  │                          │   metrics, diag)  │
  │                          └────────┬─────────┘
  │                                   │
  │                    ┌──────────────┼──────────────┐
  │                    ▼              ▼              ▼
  │               Export         LLM Explain     Reporting
  │               (xlsx/csv/     (optional)      (narrative/
  │                json/pdf)                      summary)
  │
  ├─ NL Config ("make it aggressive") ──→ LLM ──→ Config Patch
  │
  └─ CI/CD ──→ GitHub Actions ──→ Agent System ──→ Automated PR work
```
