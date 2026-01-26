# System Architecture Diagram

This document provides a comprehensive visual representation of the Trend Model Project architecture.

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              USER INTERFACES                                  │
├───────────────────┬───────────────────┬───────────────────┬──────────────────┤
│       CLI         │   Streamlit UI    │    LLM Proxy      │    Notebooks     │
│  (src/trend/      │  (streamlit_app/) │ (llm_proxy/)      │   (notebooks/)   │
│   cli.py)         │   (port 8501)     │  (port 8080)      │                  │
└────────┬──────────┴────────┬──────────┴────────┬──────────┴────────┬─────────┘
         │                   │                   │                   │
         └───────────────────┴───────────────────┴───────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           CONFIGURATION LAYER                                 │
├──────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────┐   │
│  │  YAML Configs   │  │ Pydantic Models │  │  LLM Config Generation      │   │
│  │  (config/*.yml) │  │ (config/model.py│  │  (trend nl "...")           │   │
│  │                 │  │  config/models) │  │  (llm/chain.py)             │   │
│  └────────┬────────┘  └────────┬────────┘  └─────────────┬───────────────┘   │
│           │                    │                         │                   │
│           └────────────────────┴─────────────────────────┘                   │
│                                │                                             │
│                    ┌───────────▼───────────┐                                 │
│                    │  Config Validation    │                                 │
│                    │  (config/validation,  │                                 │
│                    │   schema_validation)  │                                 │
│                    └───────────────────────┘                                 │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                            CORE ANALYSIS ENGINE                              │
│                         (src/trend_analysis/)                                │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │                        PIPELINE ORCHESTRATION                        │     │
│  │         (pipeline.py, pipeline_runner.py, pipeline_helpers.py)       │     │
│  └──────────────────────────────┬──────────────────────────────────────┘     │
│                                 │                                            │
│     ┌───────────────────────────┼───────────────────────────┐                │
│     ▼                           ▼                           ▼                │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────────┐        │
│  │ PREPROCESSING│    │    SELECTION     │    │     PORTFOLIO        │        │
│  │    STAGE     │───▶│      STAGE       │───▶│       STAGE          │        │
│  │(stages/      │    │(stages/          │    │(stages/              │        │
│  │preprocessing)│    │ selection.py)    │    │ portfolio.py)        │        │
│  └──────────────┘    └──────────────────┘    └──────────────────────┘        │
│         │                    │                        │                      │
│         ▼                    ▼                        ▼                      │
│  - Frequency detect   - Risk-free column      - Weight computation           │
│  - Missing values     - Universe filtering    - Portfolio returns            │
│  - Calendar align     - Fund ranking          - Metric aggregation           │
│  - Sample windows     - Top-N selection       - Constraint enforcement       │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           COMPUTATION MODULES                                │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐                  │
│  │    SIGNALS     │  │    METRICS     │  │   WEIGHTING    │                  │
│  │  (signals.py)  │  │  (metrics.py)  │  │   (weights/)   │                  │
│  ├────────────────┤  ├────────────────┤  ├────────────────┤                  │
│  │ - SMA/EMA      │  │ - annual_return│  │ - Equal Weight │                  │
│  │ - Momentum     │  │ - sharpe_ratio │  │ - Risk Parity  │                  │
│  │ - TrendSpec    │  │ - sortino_ratio│  │ - HRP          │                  │
│  │ - Crossover    │  │ - max_drawdown │  │ - ERC          │                  │
│  └────────────────┘  │ - volatility   │  │ - Robust       │                  │
│                      │ - info_ratio   │  └────────────────┘                  │
│                      └────────────────┘                                      │
│                                                                              │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐                  │
│  │  RISK CONTROL  │  │  BACKTESTING   │  │  MULTI-PERIOD  │                  │
│  │   (risk.py)    │  │  (backtesting/)│  │ (multi_period/)│                  │
│  ├────────────────┤  ├────────────────┤  ├────────────────┤                  │
│  │ - Vol target   │  │ - harness.py   │  │ - Period loop  │                  │
│  │ - Constraints  │  │ - bootstrap.py │  │ - Rebalancing  │                  │
│  │ - Max weight   │  │ - Walk-forward │  │ - Aggregation  │                  │
│  │ - Turnover cap │  │ - Cost model   │  │ - Threshold    │                  │
│  └────────────────┘  └────────────────┘  └────────────────┘                  │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                          EXPORT & REPORTING                                  │
│                         (export/, trend/reporting/)                          │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│   │  Excel   │  │   CSV    │  │   JSON   │  │   HTML   │  │   PDF    │       │
│   │ openpyxl │  │          │  │          │  │ Tearsheet│  │  fpdf2   │       │
│   │ xlsxwriter│ │          │  │          │  │          │  │          │       │
│   └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
│                                                                              │
│   ┌──────────────────────────────────────────────────────────────────┐       │
│   │              Unified Report Generation                            │       │
│   │        (trend/reporting/unified.py, quick_summary.py)             │       │
│   └──────────────────────────────────────────────────────────────────┘       │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

## Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              INPUT SOURCES                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐         │
│   │   CSV/Excel     │    │   YAML Config   │    │  Natural Lang   │         │
│   │  Returns Data   │    │  (defaults.yml) │    │  (trend nl ...) │         │
│   └────────┬────────┘    └────────┬────────┘    └────────┬────────┘         │
│            │                      │                      │                   │
└────────────┼──────────────────────┼──────────────────────┼───────────────────┘
             │                      │                      │
             ▼                      ▼                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              VALIDATION                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐         │
│   │  Data Validation│    │ Config Schema   │    │   LLM Chain     │         │
│   │  (data.py,      │    │  Validation     │    │  ConfigPatch    │         │
│   │   validation.py)│    │  (Pydantic)     │    │  Generation     │         │
│   └────────┬────────┘    └────────┬────────┘    └────────┬────────┘         │
│            │                      │                      │                   │
└────────────┼──────────────────────┼──────────────────────┼───────────────────┘
             │                      │                      │
             └──────────────────────┴──────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           PIPELINE STAGES                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      1. PREPROCESSING                                │   │
│   │   - detect_frequency(): Identify data frequency (daily/monthly)      │   │
│   │   - apply_missing_policy(): Handle missing values (ffill/drop/zero)  │   │
│   │   - align_calendar(): Align to calendar boundaries                   │   │
│   │   - _build_sample_windows(): Create in-sample/out-sample splits      │   │
│   └─────────────────────────────────┬───────────────────────────────────┘   │
│                                     │                                        │
│                                     ▼                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      2. SELECTION                                    │   │
│   │   - _resolve_risk_free_column(): Identify risk-free asset            │   │
│   │   - identify_risk_free_fund(): Auto-detect T-Bill or similar         │   │
│   │   - rank_select_funds(): Rank and select top-N funds                 │   │
│   │   - get_window_metric_bundle(): Compute metrics for ranking          │   │
│   └─────────────────────────────────┬───────────────────────────────────┘   │
│                                     │                                        │
│                                     ▼                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      3. PORTFOLIO                                    │   │
│   │   - compute_trend_signals(): Generate trend signals (TrendSpec)      │   │
│   │   - compute_constrained_weights(): Apply weight constraints          │   │
│   │   - calc_portfolio_returns(): Calculate weighted returns             │   │
│   │   - _compute_stats(): Compute IS/OOS performance metrics             │   │
│   └─────────────────────────────────┬───────────────────────────────────┘   │
│                                     │                                        │
└─────────────────────────────────────┼────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           OUTPUT GENERATION                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│   │   RunResult │  │   Metrics   │  │   Export    │  │   Reports   │        │
│   │   (api.py)  │  │  DataFrame  │  │   Files     │  │   HTML/PDF  │        │
│   └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Component Dependency Graph

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            USER INTERFACES                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│   │    CLI      │    │  Streamlit  │    │  LLM Proxy  │    │  Notebooks  │  │
│   │ src/trend/  │    │ streamlit_  │    │ llm_proxy/  │    │ notebooks/  │  │
│   │  cli.py     │    │   app/      │    │  server.py  │    │             │  │
│   └──────┬──────┘    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘  │
│          │                  │                  │                  │          │
└──────────┼──────────────────┼──────────────────┼──────────────────┼──────────┘
           │                  │                  │                  │
           └──────────────────┴────────┬─────────┴──────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            PUBLIC API LAYER                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌───────────────────────────────────────────────────────────────────┐     │
│   │                      api.py                                        │     │
│   │   - run_simulation(config, returns) -> RunResult                   │     │
│   │   - RunResult: metrics, details, seed, environment, analysis       │     │
│   └───────────────────────────────────┬───────────────────────────────┘     │
│                                       │                                      │
└───────────────────────────────────────┼──────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            CORE ENGINE                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────┐         ┌─────────────────────────┐           │
│   │       pipeline.py       │         │       config/           │           │
│   │  - run()                │◄───────▶│  - model.py             │           │
│   │  - run_full()           │         │  - models.py            │           │
│   │  - run_analysis()       │         │  - validation.py        │           │
│   └───────────┬─────────────┘         │  - patch.py             │           │
│               │                       │  - ui_mapping.py        │           │
│               │                       └─────────────────────────┘           │
│               ▼                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                          stages/                                     │   │
│   │   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐               │   │
│   │   │preprocessing│──▶│  selection  │──▶│  portfolio  │               │   │
│   │   └─────────────┘   └─────────────┘   └─────────────┘               │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
                    │                           │
                    ▼                           ▼
┌───────────────────────────────┐  ┌───────────────────────────────────────────┐
│     COMPUTATION MODULES       │  │          SUPPORT SERVICES                  │
├───────────────────────────────┤  ├───────────────────────────────────────────┤
│                               │  │                                            │
│  ┌─────────┐  ┌─────────┐    │  │  ┌─────────────┐  ┌─────────────┐          │
│  │ signals │  │ metrics │    │  │  │    llm/     │  │   export/   │          │
│  └─────────┘  └─────────┘    │  │  │  - chain.py │  │  - bundle.py│          │
│                               │  │  │  - providers│  │  - formats  │          │
│  ┌─────────┐  ┌─────────┐    │  │  └─────────────┘  └─────────────┘          │
│  │ weights │  │  risk   │    │  │                                            │
│  └─────────┘  └─────────┘    │  │  ┌─────────────┐  ┌─────────────┐          │
│                               │  │  │  reporting/ │  │   perf/     │          │
│  ┌─────────┐  ┌─────────┐    │  │  │  - unified  │  │ - cache.py  │          │
│  │backtest │  │ engine/ │    │  │  │  - summary  │  │ - timing    │          │
│  └─────────┘  └─────────┘    │  │  └─────────────┘  └─────────────┘          │
│                               │  │                                            │
└───────────────────────────────┘  └───────────────────────────────────────────┘
```

## Actual Module Structure

```
src/
├── trend/                              # Unified CLI Entry Point
│   ├── cli.py                          # Main CLI (trend run, trend app, trend nl, etc.)
│   ├── config_schema.py                # Core config loading
│   ├── validation.py                   # Input validation utilities
│   ├── input_validation.py             # Data input validation
│   ├── diagnostics.py                  # Diagnostic utilities
│   ├── compat_entrypoints.py           # Legacy command compatibility
│   └── reporting/
│       ├── unified.py                  # Unified HTML report generation
│       └── quick_summary.py            # Quick report generation
│
├── trend_analysis/                     # Core Analysis Engine
│   │
│   ├── api.py                          # Public API: run_simulation(), RunResult
│   ├── pipeline.py                     # Main pipeline orchestration
│   ├── pipeline_runner.py              # Pipeline execution with diagnostics
│   ├── pipeline_helpers.py             # Helper utilities for pipeline
│   ├── data.py                         # Data loading (load_csv, identify_risk_free_fund)
│   ├── cli.py                          # Legacy CLI handler
│   │
│   ├── stages/                         # Pipeline Stages
│   │   ├── preprocessing.py            # Data preparation, frequency detection
│   │   ├── selection.py                # Fund selection, ranking, risk-free resolution
│   │   └── portfolio.py                # Portfolio construction, weights, returns
│   │
│   ├── config/                         # Configuration System
│   │   ├── model.py                    # Config model definition
│   │   ├── models.py                   # Pydantic config models
│   │   ├── validation.py               # Config validation
│   │   ├── schema_validation.py        # JSON schema validation
│   │   ├── patch.py                    # LLM config patching
│   │   ├── ui_mapping.py               # Streamlit UI state <-> config
│   │   ├── schema_generator.py         # Dynamic schema generation
│   │   ├── bridge.py                   # Config bridge utilities
│   │   ├── legacy.py                   # Legacy config support
│   │   └── coverage.py                 # Config coverage tracking
│   │
│   ├── signals.py                      # Trend Signal Generation (TrendSpec)
│   ├── signal_presets.py               # Predefined signal configurations
│   │
│   ├── weights/                        # Portfolio Weighting Methods
│   │   ├── __init__.py                 # Weight computation dispatcher
│   │   ├── risk_parity.py              # Risk parity weighting
│   │   ├── hierarchical_risk_parity.py # HRP weighting
│   │   ├── equal_risk_contribution.py  # ERC weighting
│   │   ├── robust_weighting.py         # Robust weighting methods
│   │   └── robust_config.py            # Robustness configuration
│   │
│   ├── risk.py                         # Risk Control (vol targeting, constraints)
│   ├── weighting.py                    # Weight application utilities
│   ├── portfolio.py                    # Portfolio weight policies
│   │
│   ├── core/                           # Core Computation
│   │   ├── rank_selection.py           # Fund ranking and selection logic
│   │   └── metric_cache.py             # Metric computation caching
│   │
│   ├── backtesting/                    # Backtesting Engine
│   │   ├── harness.py                  # Walk-forward backtesting harness
│   │   └── bootstrap.py                # Bootstrap resampling
│   │
│   ├── engine/                         # Optimization Engine
│   │   ├── optimizer.py                # Parameter optimization
│   │   └── walkforward.py              # Walk-forward testing
│   │
│   ├── rebalancing/                    # Rebalancing Strategies
│   │   └── strategies.py               # Rebalancing strategy implementations
│   │
│   ├── llm/                            # LLM Integration
│   │   ├── chain.py                    # LangChain pipeline for config patches
│   │   ├── providers.py                # LLM provider abstractions
│   │   ├── prompts.py                  # Prompt templates
│   │   ├── validation.py               # LLM response validation
│   │   ├── result_validation.py        # Result claim validation
│   │   ├── result_feedback.py          # Feedback generation
│   │   ├── result_metrics.py           # Metric extraction
│   │   ├── schema.py                   # Schema utilities
│   │   ├── tracing.py                  # LangSmith tracing
│   │   ├── replay.py                   # NL operation replay
│   │   ├── nl_logging.py               # NL operation logging
│   │   └── injection.py                # Config injection
│   │
│   ├── llm_proxy/                      # LLM Proxy Server
│   │   ├── server.py                   # OpenAI-compatible proxy server
│   │   ├── cli.py                      # Proxy CLI
│   │   └── __main__.py                 # Module entry point
│   │
│   ├── export/                         # Export Formatters
│   │   ├── __init__.py                 # Multi-format export (Excel, CSV, JSON, etc.)
│   │   └── bundle.py                   # Reproducibility bundle export
│   │
│   ├── perf/                           # Performance Utilities
│   │   ├── rolling_cache.py            # Rolling computation cache
│   │   ├── cache.py                    # General caching
│   │   └── timing.py                   # Performance timing
│   │
│   ├── util/                           # Utility Modules
│   │   ├── frequency.py                # Frequency detection
│   │   ├── missing.py                  # Missing data handling
│   │   ├── risk_free.py                # Risk-free rate utilities
│   │   └── weights.py                  # Weight normalization
│   │
│   ├── ui/                             # UI Utilities
│   │   └── rank_widgets.py             # Ranking widgets
│   │
│   ├── diagnostics.py                  # Pipeline diagnostics
│   ├── regimes.py                      # Market regime detection
│   ├── schedules.py                    # Scheduling utilities
│   ├── timefreq.py                     # Time/frequency utilities
│   ├── time_utils.py                   # Calendar alignment
│   ├── constants.py                    # Global constants
│   ├── walk_forward.py                 # Walk-forward analysis API
│   ├── universe_catalog.py             # Universe definitions
│   └── presets.py                      # Strategy presets
│
├── trend_model/                        # Model Specification
│   ├── spec.py                         # Run specification
│   ├── app.py                          # App utilities
│   └── cli.py                          # Model CLI
│
├── backtest/                           # Backtesting Utilities
│   └── utils.py                        # Backtest helpers
│
├── data/                               # Data Contracts
│   └── contracts.py                    # Data validation contracts
│
├── analysis/                           # Analysis Results
│   └── __init__.py                     # Results class
│
└── utils/                              # General Utilities
    └── paths.py                        # Path resolution
```

## Streamlit Application Structure

```
streamlit_app/
├── app.py                              # Main entry point (Demo + Custom Analysis)
├── state.py                            # Session state management
├── config_bridge.py                    # Config <-> UI state bridge
│
├── pages/                              # Multipage App
│   ├── 1_Data.py                       # Data upload & preview
│   ├── 2_Model.py                      # Configuration UI
│   ├── 3_Results.py                    # Analysis results display
│   ├── 4_Help.py                       # Documentation & help
│   └── 8_Validation.py                 # Result validation
│
└── components/                         # Reusable Components
    ├── charts.py                       # Visualization components
    ├── comparison.py                   # Strategy comparison
    ├── comparison_export.py            # Comparison export
    ├── comparison_llm.py               # LLM-powered comparison
    ├── analysis_runner.py              # Analysis execution
    ├── demo_runner.py                  # Demo preset execution
    ├── csv_validation.py               # CSV validation
    ├── data_cache.py                   # Data caching
    ├── data_schema.py                  # Data schema validation
    ├── date_correction.py              # Date handling
    ├── disclaimer.py                   # Legal disclaimers
    ├── explain_results.py              # Result explanations
    ├── guardrails.py                   # Input guardrails
    ├── llm_settings.py                 # LLM configuration UI
    ├── policy_engine.py                # Policy evaluation
    ├── progress_eta.py                 # Progress indicators
    └── upload_guard.py                 # File upload validation
```

## Configuration Files

```
config/
├── defaults.yml                        # Base configuration schema (comprehensive)
├── demo.yml                            # Demo configuration
├── portfolio_test.yml                  # Test configuration
├── long_backtest.yml                   # Extended backtest config
├── walk_forward.yml                    # Multi-period parameters
├── robust_demo.yml                     # Robust weighting demo
├── trend_concentrated_2004.yml         # Concentrated strategy
├── trend_universe_2004.yml             # Universe strategy
│
├── presets/                            # Strategy Presets
│   ├── conservative.yml                # Low risk preset
│   ├── balanced.yml                    # Medium risk preset
│   ├── aggressive.yml                  # High risk preset
│   └── cash_constrained.yml            # Cash-constrained preset
│
└── universe/                           # Fund Universe Definitions
    ├── core.yml                        # Core fund universe
    ├── core_plus_benchmarks.yml        # Core + benchmarks
    └── managed_futures_min.yml         # Managed futures minimum
```

## Configuration Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     CONFIGURATION SOURCES                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐                  │
│   │ defaults.yml │    │  presets/    │    │ CLI Flags    │                  │
│   │   (base)     │    │ (strategy)   │    │  (override)  │                  │
│   └──────┬───────┘    └──────┬───────┘    └──────┬───────┘                  │
│          │                   │                   │                           │
│          └───────────────────┴───────────────────┘                           │
│                              │                                               │
│                              ▼                                               │
│                    ┌─────────────────┐                                       │
│                    │ load_config()   │                                       │
│                    │ (config/        │                                       │
│                    │  schema_val.)   │                                       │
│                    └────────┬────────┘                                       │
│                             │                                                │
│                             ▼                                                │
│   ┌───────────────────────────────────────────────────────────────────┐     │
│   │              NATURAL LANGUAGE INPUT (optional)                     │     │
│   │         trend nl "use risk parity with 10% vol target"             │     │
│   └───────────────────────────────────────────────────────────────────┘     │
│                             │                                                │
│                             ▼                                                │
│                    ┌─────────────────┐                                       │
│                    │ ConfigPatchChain│                                       │
│                    │   (llm/chain)   │                                       │
│                    └────────┬────────┘                                       │
│                             │                                                │
│                             ▼                                                │
│                    ┌─────────────────┐                                       │
│                    │ apply_patch()   │                                       │
│                    │ (config/patch)  │                                       │
│                    └────────┬────────┘                                       │
│                             │                                                │
│                             ▼                                                │
│                    ┌─────────────────┐                                       │
│                    │validate_config()│                                       │
│                    │  (Pydantic +    │                                       │
│                    │   JSON Schema)  │                                       │
│                    └────────┬────────┘                                       │
│                             │                                                │
│                             ▼                                                │
│                    ┌─────────────────┐                                       │
│                    │  Final Config   │                                       │
│                    │  (ConfigModel)  │                                       │
│                    └─────────────────┘                                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## External Integrations

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         EXTERNAL SERVICES                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                        LLM PROVIDERS                                 │   │
│   │  (configured via TREND_LLM_PROVIDER environment variable)            │   │
│   ├──────────────┬──────────────┬──────────────┬────────────────────────┤   │
│   │    OpenAI    │   Anthropic  │    Ollama    │   Custom Endpoint      │   │
│   │   (GPT-4o,   │   (Claude)   │   (Local)    │   (OpenAI-compatible)  │   │
│   │  GPT-4o-mini)│              │              │                        │   │
│   └──────────────┴──────────────┴──────────────┴────────────────────────┘   │
│                              │                                               │
│                              ▼                                               │
│                    ┌─────────────────┐                                       │
│                    │   LangChain     │                                       │
│                    │   Framework     │                                       │
│                    │  (>=1.2, <1.3)  │                                       │
│                    └────────┬────────┘                                       │
│                             │                                                │
│                             ▼                                                │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                   LANGSMITH (Tracing - Optional)                     │   │
│   │              (configured via LANGSMITH_API_KEY)                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Key Design Patterns

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         DESIGN PATTERNS                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Pattern                  │ Implementation           │ Location             │
│   ─────────────────────────┼──────────────────────────┼────────────────────  │
│   Pipeline                 │ Multi-stage processing   │ stages/              │
│   Strategy                 │ Pluggable weighting      │ weights/             │
│   Factory                  │ Config model creation    │ config/model.py      │
│   Facade                   │ Unified API entry        │ api.py               │
│   Chain of Responsibility  │ LLM validation chain     │ llm/chain.py         │
│   Observer                 │ Streamlit session state  │ streamlit_app/state  │
│   Template Method          │ Pipeline stages          │ stages/*.py          │
│   Decorator                │ Config coverage tracking │ config/coverage.py   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Performance Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       PERFORMANCE OPTIMIZATIONS                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌───────────────────────────────┐    ┌───────────────────────────────┐    │
│   │     Vectorized Computation    │    │      In-Memory Caching        │    │
│   │   ─────────────────────────   │    │   ─────────────────────────   │    │
│   │   - NumPy array operations    │    │   - perf/rolling_cache.py     │    │
│   │   - Pandas DataFrame ops      │    │   - perf/cache.py             │    │
│   │   - Scipy optimizations       │    │   - Dataset hash-based keys   │    │
│   └───────────────────────────────┘    └───────────────────────────────┘    │
│                                                                              │
│   ┌───────────────────────────────┐    ┌───────────────────────────────┐    │
│   │      Lazy Data Loading        │    │      Metric Caching           │    │
│   │   ─────────────────────────   │    │   ─────────────────────────   │    │
│   │   - Load data on demand       │    │   - core/metric_cache.py      │    │
│   │   - Stream large files        │    │   - Window-key based          │    │
│   │   - Memory-efficient reads    │    │   - Reuse across periods      │    │
│   └───────────────────────────────┘    └───────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## CLI Command Structure

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CLI COMMANDS (trend)                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   trend run                                                                  │
│   ├── -c, --config         Path to YAML config                              │
│   ├── -i, --input          Override returns CSV path                        │
│   ├── --seed               Force random seed                                │
│   ├── --bundle             Write reproducibility bundle                     │
│   ├── --universe           Select named universe                            │
│   ├── --preset             Apply trend preset                               │
│   └── --config-coverage    Report config key usage                          │
│                                                                              │
│   trend report                                                               │
│   ├── -c, --config         Path to YAML config                              │
│   ├── --out                Directory for artefacts                          │
│   ├── --output             Path to HTML report                              │
│   ├── --formats            Export formats (csv, json, xlsx, txt)            │
│   └── --pdf                Also generate PDF report                         │
│                                                                              │
│   trend nl <instruction>                                                     │
│   ├── --in                 Input config file                                │
│   ├── --out                Output config file                               │
│   ├── --diff               Show unified diff                                │
│   ├── --dry-run            Print without writing                            │
│   ├── --run                Validate and run pipeline                        │
│   ├── --provider           LLM provider (openai, anthropic, ollama)         │
│   ├── --model              Override LLM model                               │
│   └── --temperature        Override LLM temperature                         │
│                                                                              │
│   trend app                Launch Streamlit UI (port 8501)                  │
│   trend check              Print environment info                            │
│   trend quick-report       Build compact HTML from artefacts                │
│   trend explain            Explain results with LLM                          │
│   trend stress             Run stress scenario analysis                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Quick Reference

### Entry Points

| Command | Description | Port |
|---------|-------------|------|
| `trend run -c config.yml` | Run analysis pipeline | - |
| `trend app` | Launch Streamlit UI | 8501 |
| `trend nl "..."` | Natural language config edit | - |
| `trend report` | Generate reports | - |
| `trend-llm-proxy` | Start LLM proxy server | 8080 |

### Key Configuration Files

| File | Purpose |
|------|---------|
| `config/defaults.yml` | Base configuration with all options |
| `config/presets/balanced.yml` | Balanced risk strategy preset |
| `config/presets/conservative.yml` | Conservative strategy preset |
| `config/presets/aggressive.yml` | Aggressive strategy preset |
| `config/universe/core.yml` | Core fund universe definition |
| `pyproject.toml` | Package dependencies and metadata |

### Environment Variables

```bash
# LLM Configuration
TREND_LLM_PROVIDER="openai"          # LLM provider (openai, anthropic, ollama)
TREND_LLM_MODEL="gpt-4o-mini"        # LLM model selection
TREND_LLM_TEMPERATURE="0.2"          # LLM temperature
TREND_LLM_API_KEY="..."              # API key (or use provider-specific)
OPENAI_API_KEY="..."                 # OpenAI API key
ANTHROPIC_API_KEY="..."              # Anthropic API key
LANGSMITH_API_KEY="..."              # LangSmith tracing (optional)
LANGCHAIN_PROJECT="trend"            # LangSmith project name

# Runtime Configuration
TREND_SEED="42"                      # Random seed override
TREND_DISABLE_PERF_LOGS="false"      # Disable performance logging
TREND_FORCE_LEGACY_CLI="false"       # Force legacy CLI behavior
```

### Core Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| numpy | 2.4.1 | Numerical computation |
| pandas | 2.3.3 | Data manipulation |
| scipy | 1.17.0 | Scientific computing |
| pydantic | 2.12.5 | Config validation |
| PyYAML | 6.0.3 | YAML parsing |
| streamlit | 1.53.1 | Web UI (optional) |
| langchain | >=1.2,<1.3 | LLM integration (optional) |
