<div align="center">

# 🤖 AutoCodeRover v2 — Interactive

**An Autonomous, Multi-Agent System for Automated Program Repair Using LLMs**

[![Python 3.11+](https://img.shields.io/badge/Python-3.11%2B-blue?logo=python&logoColor=white)](https://python.org)
[![License: AGPL v3](https://img.shields.io/badge/License-AGPL%20v3-blue.svg)](LICENSE)
[![SWE-bench](https://img.shields.io/badge/Benchmark-SWE--bench-green)](https://www.swebench.com/)

*AutoCodeRover-Interactive is a fully autonomous software engineering agent that understands bug reports, searches codebases for relevant context, generates reproducer tests, writes patches, reviews and validates them, and self-corrects — all with optional human-in-the-loop feedback at every stage.*

</div>

---

## Table of Contents

- [Overview](#overview)
- [System Architecture](#system-architecture)
  - [High-Level Pipeline](#high-level-pipeline)
  - [Multi-Agent Orchestration](#multi-agent-orchestration)
  - [Action / State Machine](#action--state-machine)
  - [Agent Layer](#agent-layer)
  - [Model Abstraction Layer](#model-abstraction-layer)
  - [Multi-Language Code Search Backend](#multi-language-code-search-backend)
  - [Patch Generation & Application](#patch-generation--application)
  - [Interactive Replay & Feedback Loop](#interactive-replay--feedback-loop)
- [Supported LLM Providers](#supported-llm-providers)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Installation — Docker (Recommended)](#installation--docker-recommended)
  - [Installation — Local (Conda)](#installation--local-conda)
  - [Environment Variables](#environment-variables)
- [Usage](#usage)
  - [Execution Modes](#execution-modes)
  - [GitHub Issue Mode](#github-issue-mode)
  - [Local Issue Mode](#local-issue-mode)
  - [SWE-bench Mode](#swe-bench-mode)
  - [Replay Mode (Interactive Feedback)](#replay-mode-interactive-feedback)
- [Advanced Configuration](#advanced-configuration)
  - [Model Map — Per-Agent Model Assignment](#model-map--per-agent-model-assignment)
  - [Config File for Batch Experiments](#config-file-for-batch-experiments)
  - [Feature Flags & Runtime Options](#feature-flags--runtime-options)
- [Project Structure](#project-structure)
- [Output Artifacts](#output-artifacts)
- [License](#license)
- [Contact](#contact)

---

## Overview

AutoCodeRover v2 Interactive is a research-grade, production-capable system for **automated program repair (APR)**. Given a bug report (from GitHub, a local file, or SWE-bench), the system autonomously:

1. **Reproduces** the bug by generating a test that triggers the failure.
2. **Localizes** the fault using LLM-driven code search and optional Spectrum-Based Fault Localization (SBFL).
3. **Generates** a candidate patch via a dedicated patch-writing agent with iterative refinement.
4. **Reviews** the patch using an independent reviewer agent that cross-validates the patch against the reproducer test.
5. **Validates** the patch against existing regression test suites.
6. **Self-corrects** — if a patch is inapplicable, a self-fix agent diagnoses the root cause and replays the pipeline from the most suspicious stage.
7. **Selects** the best patch from multiple candidates using a majority-vote selection agent.

The **Interactive** variant introduces a **human-in-the-loop feedback mechanism** — engineers can inject feedback at the context retrieval or patch generation stages, and the system replays from that point with the feedback incorporated into subsequent LLM prompts.

---

## System Architecture

### High-Level Pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│                         INPUT                                       │
│   GitHub Issue  |  Local Issue File  |  SWE-bench Task Instance     │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    META-AGENT (Orchestrator)                         │
│                                                                     │
│  HardcodedMetaAgent (default) or MetaAgent (LLM-driven)            │
│  Controls the execution order of actions via a generator-based loop │
└────────────────────────────┬────────────────────────────────────────┘
                             │
        ┌────────────────────┼─────────────────────┐
        ▼                    ▼                     ▼
 ┌──────────────┐   ┌───────────────┐   ┌─────────────────┐
 │  Generate    │   │  Generate     │   │   Context       │
 │  Reproducer  │   │  SBFL         │   │   Retrieval     │
 │  Test        │   │  (optional)   │   │   (Code Search) │
 └──────┬───────┘   └───────┬───────┘   └────────┬────────┘
        │                    │                     │
        │          ┌─────────┴─────────┐           │
        └──────────┤   State Machine   ├───────────┘
                   │   (ActionState)   │
                   └─────────┬─────────┘
                             │
                ┌────────────┼────────────┐
                ▼            ▼            ▼
         ┌────────────┐ ┌──────────┐ ┌──────────┐
         │  Write     │ │  Review  │ │  Self    │
         │  Patch     │ │  Patch   │ │  Fix     │
         └─────┬──────┘ └─────┬────┘ └─────┬────┘
               │              │             │
               └──────────────┼─────────────┘
                              ▼
                    ┌──────────────────┐
                    │   Select Patch   │
                    │   (Best-of-N)    │
                    └────────┬─────────┘
                             ▼
                    ┌──────────────────┐
                    │  selected_patch  │
                    │     .json        │
                    └──────────────────┘
```

### Multi-Agent Orchestration

The system uses two orchestration strategies:

| Strategy | Class | Description |
|----------|-------|-------------|
| **Hardcoded** | `HardcodedMetaAgent` | Executes a deterministic sequence: Reproduce → SBFL → Search → Patch → Review → Self-Fix. Default mode. |
| **LLM-Driven** | `MetaAgent` | Uses an LLM to dynamically decide the next action based on accumulated state. Enabled with `--enable-meta-agent`. |

Both implement a **Python generator-based coroutine loop** — each action yields its type and receives a state-change description, enabling the orchestrator to make informed decisions about the next step.

### Action / State Machine

Every stage of the pipeline is modeled as an **Action** — an immutable, serializable unit of work:

| Action | Purpose |
|--------|---------|
| `GenerateReproducer` | Generates a reproducing test for the bug using a dedicated `TestAgent`. |
| `GenerateSBFL` | Runs Spectrum-Based Fault Localization to identify suspicious code regions. |
| `ContextRetrieval` | Drives the `SearchAgent` to iteratively search the codebase for bug-relevant locations. |
| `WritePatch` | Invokes the `PatchAgent` to generate a candidate patch, validates applicability. |
| `ReviewPatch` | Cross-validates the patch against the reproducer using the `ReviewerAgent`. |
| `SelfFix` | Diagnoses why a patch was inapplicable and replays from the identified problematic stage. |
| `SelectPatch` | Aggregates patches across retries, runs regression validation, and selects the best candidate. |
| `TerminalExecution` | Executes shell commands via an LLM-driven terminal agent (for exploratory debugging). |
| `FeedbackContext` | Re-runs context retrieval incorporating human feedback. |
| `FeedbackPatch` | Re-runs patch generation incorporating human feedback. |

All actions share a **frozen, immutable `ActionState`** dataclass that captures the full pipeline state — bug locations, message threads, patch indices, test indices, reproduction results, and review maps. State transitions produce new `ActionState` instances (functional paradigm), ensuring clean reproducibility.

Every action execution is **recorded to JSON** (`recording/*.json`), enabling full deterministic replay and interactive feedback injection.

### Agent Layer

Each action delegates to one or more specialized agents:

| Agent | Role |
|-------|------|
| `SearchFrontend` + `SearchAgent` | Multi-round conversational code search — the LLM issues API-style search calls (e.g., `search_method`, `search_class`, `search_code`) parsed by a `ProxyAgent` and executed against the `SearchBackend`. |
| `PatchAgent` | Constructs a patch from the search context, applies iterative retries with temperature escalation, validates diff applicability. |
| `TestAgent` (Reproducer) | Generates a test that triggers the reported bug, validates reproduction, supports feedback-driven refinement. |
| `ReviewerAgent` | Executes the reproducer test with and without the patch, then asks an LLM to judge correctness of both the patch and the test. |
| `SelfFixAgent` | Analyzes why a patch was inapplicable through a 4-step chain: collect reasons → analyze → identify responsible agent → generate corrective feedback. |
| `ProxyAgent` | Converts free-form LLM search responses into structured JSON API calls. |
| `TerminalAgent` | Executes shell commands with timeout protection, captures output, and uses an LLM to review results and decide next steps. |
| `TestBlacklistAgent` | Identifies tests that are irrelevant to the issue and should be excluded from validation. |

All agents use a `MessageThread` abstraction for conversation management, supporting serialization, template substitution, and tool-call interleaving.

### Model Abstraction Layer

The system provides a **provider-agnostic model layer** with auto-registration:

```
app/model/
├── common.py          # Abstract Model base class, cost tracking, MODEL_HUB registry
├── register.py        # Auto-discovers and registers all model classes at startup
├── litellm_model.py   # LiteLLM-based abstract model (shared by most providers)
├── gpt.py             # OpenAI GPT-4o, o1-mini, etc.
├── claude.py          # Anthropic Claude 3/3.5 (Opus, Sonnet, Haiku)
├── deepseek.py        # DeepSeek V3 / R1
├── gemini.py          # Google Gemini (via Vertex AI or API key)
├── bedrock.py         # AWS Bedrock (Claude, etc.)
├── openrouter.py      # OpenRouter (multi-provider routing)
├── ollama.py          # Local models via Ollama (Llama 3, etc.)
└── tools/             # Tool-use implementations (file, search, terminal, browser)
```

Key design choices:
- **Singleton model registry** (`MODEL_HUB`) populated at startup via reflection-based auto-discovery.
- **Per-process cost accumulation** with configurable cost limits (`--cost-limit`).
- **Per-agent model assignment** via `--model-map` (e.g., use Claude for search, GPT-4o for patching).
- **Automatic retry with exponential backoff** for transient API failures.
- **Multimodal support** — image content in issue descriptions is converted to base64 data URIs.

### Multi-Language Code Search Backend

The search backend uses **tree-sitter** parsers for language-aware structural code search:

| Language | Supported Search Operations |
|----------|-----------------------------|
| Python | Classes, methods, functions, code snippets, top-level definitions |
| Java | Classes, methods, fields, imports |
| JavaScript / TypeScript | Functions, classes, methods, modules |
| C / C++ | Functions, structs, headers |
| Go | Functions, types, interfaces |
| PHP | Classes, functions, methods |
| Text / Config | Plain-text search for non-code files |

Search APIs available to the LLM include: `search_class`, `search_method`, `search_method_in_class`, `search_method_in_file`, `search_class_in_file`, `search_code`, `search_code_in_file`, and more — each backed by indexed ASTs for fast lookup.

### Patch Generation & Application

The patching subsystem supports multi-language diff generation:

1. The `PatchAgent` asks the LLM to produce search/replace edit blocks.
2. `patch_utils.py` parses these blocks and attempts fuzzy matching against the original source (stripping line numbers, normalizing whitespace).
3. A language-specific `PatchService` validates the result (e.g., Python AST lint check via `ast.parse`).
4. Successfully applied patches are registered in the immutable `PatchIndex` with their raw response, diff content, and feedback history.
5. Validation runs the project's test suite (via Docker containers for SWE-bench tasks) to check for regressions.

### Interactive Replay & Feedback Loop

The **replay system** is a first-class feature enabling human-in-the-loop interaction:

1. Every action execution is serialized to `recording/*.json`, capturing full state transitions.
2. An engineer reviews the output and injects **feedback** (e.g., "the search missed the relevant module" or "the patch should modify `utils.py` instead").
3. The `replay` subcommand replays the pipeline up to the feedback point, then re-executes subsequent actions with the feedback injected into LLM prompts.
4. Feedback can target either **Context Retrieval** or **Patch Generation** stages.
5. The `SelfFix` action automates this loop — when a patch is inapplicable, it uses an LLM to diagnose which stage failed and generates corrective feedback autonomously.

---

## Supported LLM Providers

| Provider | Models | Auth Env Variable |
|----------|--------|-------------------|
| **OpenAI** | GPT-4o (multiple dates), o1-mini | `OPENAI_API_KEY` |
| **Anthropic** | Claude 3 Opus, Claude 3/3.5 Sonnet, Claude 3 Haiku | `ANTHROPIC_API_KEY` |
| **Google** | Gemini Pro, Gemini Flash | `GEMINI_API_KEY` or `GOOGLE_APPLICATION_CREDENTIALS` |
| **DeepSeek** | DeepSeek V3, DeepSeek R1 | `DEEPSEEK_API_KEY` or `OPENAI_API_KEY` |
| **AWS Bedrock** | Any Bedrock-supported model | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION_NAME` |
| **OpenRouter** | Any OpenRouter model | `OPENROUTER_API_KEY` |
| **Ollama** (local) | Llama 3 (8B/70B), Llama Vision | No key required |

---

## Getting Started

### Prerequisites

- **Python 3.11+**
- **Conda** (Miniconda or Anaconda)
- **Docker** (for SWE-bench mode and Docker-based setup)
- **Git**
- An API key for at least one supported LLM provider

### Installation — Docker (Recommended)

```bash
# Set your API key(s)
export OPENAI_API_KEY=sk-YOUR-OPENAI-API-KEY-HERE
# Or for Anthropic:
export ANTHROPIC_API_KEY=sk-ant-api...

# Build the image
docker build -f Dockerfile -t acr .

# Run interactively
docker run -it -e OPENAI_API_KEY="${OPENAI_API_KEY}" acr
```

### Installation — Local (Conda)

```bash
# Clone the repository
git clone <repo-url> && cd acr-interactive

# Create the conda environment
conda env create -f environment.yml
conda activate auto-code-rover

# Or install via pip
pip install -r requirements.txt
pip install -r requirements-languages.txt  # tree-sitter parsers
```

### Environment Variables

| Variable | Required For | Description |
|----------|-------------|-------------|
| `OPENAI_API_KEY` | OpenAI models | OpenAI API key |
| `ANTHROPIC_API_KEY` | Claude models | Anthropic API key |
| `GEMINI_API_KEY` | Gemini models | Google Gemini API key |
| `GOOGLE_APPLICATION_CREDENTIALS` | Vertex AI | Path to GCP service account JSON |
| `DEEPSEEK_API_KEY` | DeepSeek models | DeepSeek API key |
| `AWS_ACCESS_KEY_ID` | Bedrock models | AWS access key |
| `AWS_SECRET_ACCESS_KEY` | Bedrock models | AWS secret key |
| `AWS_REGION_NAME` | Bedrock models | AWS region (e.g., `us-east-1`) |
| `OPENROUTER_API_KEY` | OpenRouter models | OpenRouter API key |

---

## Usage

### Execution Modes

```
PYTHONPATH=. python app/main.py <mode> [options]
```

| Mode | Description |
|------|-------------|
| `github-issue` | Resolve a live GitHub issue by cloning the repo and fetching the issue body. |
| `local-issue` | Resolve a bug described in a local file against a local repository. |
| `swe-bench` | Run on SWE-bench task instances (used for benchmarking and evaluation). |
| `scripted` | Run on a pre-configured project with metadata and bug report files. |
| `replay` | Replay a previous session with injected human feedback. |
| `extract-patches` | Post-process experiment outputs to extract all patches. |

### GitHub Issue Mode

Resolve a live GitHub issue end-to-end:

```bash
cd /opt/auto-code-rover
conda activate auto-code-rover

PYTHONPATH=. python app/main.py github-issue \
  --output-dir output \
  --setup-dir setup \
  --model gpt-4o-2024-05-13 \
  --model-temperature 0.2 \
  --task-id <task-id> \
  --clone-link <git-clone-url> \
  --commit-hash <commit-hash> \
  --issue-link <github-issue-url>
```

**Example** — resolving a LangChain issue:

```bash
PYTHONPATH=. python app/main.py github-issue \
  --output-dir output \
  --setup-dir setup \
  --model gpt-4o-2024-05-13 \
  --model-temperature 0.2 \
  --task-id langchain-20453 \
  --clone-link https://github.com/langchain-ai/langchain.git \
  --commit-hash cb6e5e5 \
  --issue-link https://github.com/langchain-ai/langchain/issues/20453
```

| Parameter | Description |
|-----------|-------------|
| `--task-id` | A unique string identifier for this issue (your choice). |
| `--clone-link` | Git clone URL for the target repository. |
| `--commit-hash` | The commit to checkout (the version containing the bug). If omitted, uses HEAD. |
| `--issue-link` | URL of the GitHub issue page. |
| `--setup-dir` | Directory where repositories will be cloned. |
| `--use-comments` | Include issue comments in the problem statement. |

### Local Issue Mode

Resolve a bug described in a local text file:

```bash
PYTHONPATH=. python app/main.py local-issue \
  --output-dir output \
  --model gpt-4o-2024-05-13 \
  --model-temperature 0.2 \
  --task-id my-bug-fix \
  --local-repo /path/to/local/project \
  --issue-file /path/to/issue_description.txt
```

### SWE-bench Mode

For standardized benchmarking against [SWE-bench](https://www.swebench.com/) task instances:

```bash
# Run a single task
PYTHONPATH=. python app/main.py swe-bench \
  --model gpt-4o-2024-05-13 \
  --output-dir output \
  --task-id django__django-11133

# Run multiple tasks from a file
PYTHONPATH=. python app/main.py swe-bench \
  --model gpt-4o-2024-05-13 \
  --output-dir output \
  --task-list-file tasks.txt \
  --num-processes 4
```

> **Note:** SWE-bench mode automatically builds Docker environment images for each task instance. Requires Docker.

See [EXPERIMENT.md](EXPERIMENT.md) for detailed experiment configuration and batch execution instructions.

### Replay Mode (Interactive Feedback)

Replay a previous session and inject human feedback:

```bash
PYTHONPATH=. python app/main.py replay <path-to-output-dir> \
  --model gpt-4o-2024-05-13 \
  --output-dir output
```

To replay up to a specific action:

```bash
PYTHONPATH=. python app/main.py replay <path-to-output-dir> \
  --replay-action 2 \
  --model gpt-4o-2024-05-13 \
  --output-dir output
```

The system reads `recording/*.json` files, detects feedback injections, and replays the pipeline from the feedback point forward.

---

## Advanced Configuration

### Model Map — Per-Agent Model Assignment

Assign different LLMs to different pipeline stages for cost optimization or quality tuning:

```bash
PYTHONPATH=. python app/main.py github-issue \
  --model gpt-4o-2024-05-13 \
  --model-map search::claude-3-5-sonnet-20241022::0.0 \
  --model-map patch::gpt-4o-2024-11-20::0.2 \
  --model-map reproducer::claude-3-5-sonnet-20241022 \
  --model-map review::gpt-4o-2024-05-13 \
  ...
```

**Format:** `<agent-class>::<model-name>[::temperature]`

| Agent Class | Maps To |
|-------------|---------|
| `search` | Context retrieval / code search agent |
| `patch` | Patch generation agent |
| `reproducer` | Reproducer test generation agent |
| `proxy` | JSON proxy agent (parses search API calls) |
| `review` | Patch reviewer agent |
| `select` | Patch selection agent |
| `terminal` | Shell command execution agent |
| `meta` | Meta-agent (when LLM-driven orchestration is enabled) |
| `self_fix` | Self-fix diagnostic agent |
| `test_select` | Test blacklist agent |

### Config File for Batch Experiments

For large-scale experiments, use a config file instead of CLI arguments:

```properties
# conf/experiment.conf
id:my_experiment
date:2025-02-20
experiment_dir:/path/to/output

model:claude-3-5-sonnet-20241022
temperature:0.0
conv_round_limit:10
selected_tasks_file:/path/to/tasks.txt
enable_validation:true
num_processes:8
```

```bash
python scripts/run.py conf/experiment.conf
```

### Feature Flags & Runtime Options

| Flag | Default | Description |
|------|---------|-------------|
| `--model` | `gpt-3.5-turbo-0125` | Primary LLM model. |
| `--model-temperature` | `0.0` | Default temperature for all agents. |
| `--conv-round-limit` | `15` | Max conversation rounds for context retrieval. |
| `--enable-sbfl` | `false` | Enable Spectrum-Based Fault Localization. |
| `--enable-validation` | `false` | Validate patches against regression tests. |
| `--enable-angelic` | `false` | Enable angelic debugging (experimental). |
| `--reproduce-and-review` | `false` | Enable the full reproduce → patch → review cycle. |
| `--num-processes` | `1` | Number of parallel task processes. |
| `--cost-limit` | `-1.0` (disabled) | Max API cost ($) before terminating a task. |
| `--record` | `true` | Record actions for subsequent replay. |
| `--no-print` | `false` | Suppress verbose console output. |

---

## Project Structure

```
acr-interactive/
├── app/
│   ├── main.py                  # Entry point — CLI parsing, task dispatch
│   ├── config.py                # Global configuration variables
│   ├── args.py                  # CLI argument definitions (ConfigArgParse)
│   ├── data.py                  # Core data models (Patch, Test, Review)
│   ├── run_task.py              # Task execution loop (retry, state management)
│   ├── raw_tasks.py             # Raw task types (GitHub, Local, SWE-bench, Scripted)
│   ├── postprocessing.py        # Patch extraction and SWE-bench input formatting
│   ├── result_analysis.py       # Experiment result analysis and reporting
│   ├── log.py                   # Logging utilities
│   ├── task_counter.py          # Task progress tracking
│   │
│   ├── actions/                 # Pipeline actions (state machine)
│   │   ├── action.py            # Abstract Action base class with record/replay
│   │   ├── context_retrieval.py # LLM-driven code search
│   │   ├── write_patch.py       # Patch generation with iterative refinement
│   │   ├── review_patch.py      # Patch + reproducer cross-validation
│   │   ├── generate_reproducer.py  # Bug-reproducing test generation
│   │   ├── generate_sbfl.py     # Spectrum-based fault localization
│   │   ├── select_patch.py      # Best-patch selection (validation + LLM vote)
│   │   ├── self_fix.py          # Self-correction for inapplicable patches
│   │   ├── terminal_execution.py   # Shell command execution via LLM
│   │   ├── feedback_context.py  # Human-in-the-loop: context retrieval feedback
│   │   ├── feedback_patch.py    # Human-in-the-loop: patch generation feedback
│   │   ├── task_execution.py    # Base class for actions that run tests
│   │   └── state/               # Immutable state models
│   │       ├── action_state.py  # Central state dataclass (frozen, slotted)
│   │       ├── patch_index.py   # Patch history, diffs, feedbacks
│   │       ├── test_index.py    # Test history and feedbacks
│   │       ├── feedback_history.py  # Feedback tracking
│   │       ├── feedback_index.py    # Agent-stage enum for feedback routing
│   │       ├── patch_selection.py   # Final selection result
│   │       └── reproduction_info.py # Reproducer execution results
│   │
│   ├── agents/                  # LLM agent implementations
│   │   ├── agent_search.py      # Search frontend (multi-round LLM conversation)
│   │   ├── agent_write_patch.py # Patch generation agent
│   │   ├── agent_reproducer.py  # Bug reproducer test agent
│   │   ├── agent_meta.py        # Meta-agent / hardcoded orchestrator
│   │   ├── agent_proxy.py       # JSON proxy (LLM response → structured API calls)
│   │   ├── agent_command.py     # Terminal command execution agent
│   │   ├── agent_self_fix.py    # Self-fix diagnostic agent
│   │   ├── agent_select.py      # Patch selection agent
│   │   ├── agent_test_blacklist.py  # Test relevance filtering
│   │   ├── reviewer.py          # Patch+test review agent
│   │   ├── message_thread.py    # Conversation thread abstraction
│   │   ├── prompts/             # 28 prompt templates (.md files)
│   │   ├── search/              # Code search backend
│   │   │   ├── search_backend.py    # Search service registry & dispatch
│   │   │   ├── search_manage.py     # Search orchestration (frontend ↔ backend)
│   │   │   ├── search_service.py    # Abstract search service interface
│   │   │   ├── python/          # Python AST-based search (tree-sitter)
│   │   │   ├── java/            # Java search implementation
│   │   │   ├── js/              # JavaScript search implementation
│   │   │   ├── ts/              # TypeScript search implementation
│   │   │   ├── cpp/             # C/C++ search implementation
│   │   │   ├── go/              # Go search implementation
│   │   │   ├── php/             # PHP search implementation
│   │   │   └── text/            # Plain-text fallback search
│   │   └── testing/             # Test execution and result types
│   │
│   ├── model/                   # LLM provider abstraction layer
│   │   ├── common.py            # Abstract Model, cost tracking, MODEL_HUB
│   │   ├── register.py          # Auto-discovery model registration
│   │   ├── litellm_model.py     # LiteLLM shared implementation
│   │   ├── gpt.py               # OpenAI models
│   │   ├── claude.py            # Anthropic models
│   │   ├── deepseek.py          # DeepSeek models
│   │   ├── gemini.py            # Google Gemini models
│   │   ├── bedrock.py           # AWS Bedrock models
│   │   ├── openrouter.py        # OpenRouter models
│   │   ├── ollama.py            # Local Ollama models
│   │   └── tools/               # Tool-use (function calling) implementations
│   │
│   ├── patch/                   # Patch parsing, application, and validation
│   │   ├── patch_utils.py       # Diff extraction, fuzzy matching, application
│   │   ├── patch_service.py     # Abstract patch service interface
│   │   └── python/              # Python-specific patch validation (AST lint)
│   │
│   ├── sbfl/                    # Spectrum-Based Fault Localization
│   │   ├── sbfl.py              # SBFL orchestration
│   │   ├── swe_bench.py         # SWE-bench SBFL integration
│   │   └── utils.py             # Coverage data collation
│   │
│   ├── tasks/                   # Task abstractions
│   │   ├── abstract_task.py     # Abstract Task (project setup, validation, cleanup)
│   │   ├── local/               # Local project tasks
│   │   ├── scripted/            # Scripted tasks (pre-configured metadata)
│   │   └── swe_bench/           # SWE-bench Docker-based tasks
│   │
│   ├── utils/                   # Shared utilities
│   │   ├── execution.py         # Command execution helpers
│   │   ├── files.py             # File I/O, directory management
│   │   └── git.py               # Git operations (clone, checkout, reset)
│   │
│   └── validation/              # Patch validation against test suites
│       └── utils.py             # Validation, angelic debugging, fix location comparison
│
├── scripts/
│   ├── run.py                   # Batch experiment driver
│   └── replay/                  # Session replay utilities
│       ├── replay.py            # Terminal-based conversation replayer
│       └── README.md            # Replay documentation
│
├── conf/                        # Configuration files
│   ├── example.conf             # Example experiment config
│   └── example_tasks.txt        # Example task list
│
├── Dockerfile                   # Production Docker image
├── Dockerfile.experiment        # Experiment Docker image
├── Dockerfile.minimal           # Minimal Docker image
├── environment.yml              # Conda environment specification
├── requirements.txt             # Core Python dependencies
├── requirements-languages.txt   # Tree-sitter language parsers
├── requirements-environment.txt # SWE-bench / Docker dependencies
├── requirements-dev.txt         # Development dependencies
├── pyproject.toml               # Project metadata
├── EXPERIMENT.md                # Experiment execution guide
└── LICENSE                      # AGPL v3 (non-commercial) / Commercial dual license
```

---

## Output Artifacts

After a successful run, the output directory contains:

| File | Description |
|------|-------------|
| `selected_patch.json` | Final selected patch path and selection reason. |
| `meta.json` | Task metadata, model configuration, and model map. |
| `cost.json` | Total API cost, input/output token counts, wall-clock time. |
| `output_0/` | First retry iteration directory. |
| `output_0/conv_patch_*.json` | Full LLM conversation thread for patch generation. |
| `output_0/patch_raw_*.md` | Raw LLM patch response (markdown). |
| `output_0/extracted_patch_*.diff` | Extracted unified diff. |
| `output_0/extract_status.json` | Patch extraction result (applicable/unmatched/etc.). |
| `output_0/search_0/` | Context retrieval search results and tool call layers. |
| `output_0/recording/*.json` | Serialized action states for replay. |
| `output_0/display.jsonl` | Structured display log for UI rendering. |
| `output_0/bug_locs.dill` | Serialized bug location objects. |

---

## License

This project is **dual-licensed**:

- **Non-commercial use** (including academic research): [GNU Affero General Public License v3](LICENSE)
- **Commercial use**: Contact the authors for a commercial license.

---

## Contact

> [!NOTE]
> For issues, questions, or collaboration inquiries:
>
> - 📧 Email: [info@autocoderover.dev](mailto:info@autocoderover.dev)
> - 💬 Discord: [AutoCodeRover Community](https://discord.com/invite/ScXsdE49JY)
