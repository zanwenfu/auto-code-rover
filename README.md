<div align="center">

# AutoCodeRover v2 — Interactive

**Autonomous Program Repair via Iterative Specification Inference, Multi-Agent Orchestration, and Human-in-the-Loop Feedback**

[![Python 3.11+](https://img.shields.io/badge/Python-3.11%2B-blue?logo=python&logoColor=white)](https://python.org)
[![License: AGPL v3](https://img.shields.io/badge/License-AGPL%20v3-blue.svg)](LICENSE)
[![SWE-bench](https://img.shields.io/badge/SWE--bench_Verified-51.6%25-brightgreen)](https://www.swebench.com/)
[![ISSTA 2024](https://img.shields.io/badge/Published-ISSTA_2024-orange)](https://doi.org/10.1145/3650212.3680384)

AutoCodeRover-Interactive is a fully autonomous software engineering agent that resolves real-world GitHub issues end-to-end — from bug reproduction through fault localization, specification inference, patch generation, review, and self-correction — with optional human feedback at every stage.

**[Paper (ISSTA 2024)](https://doi.org/10.1145/3650212.3680384)** · **[SpecRover (arXiv)](https://arxiv.org/abs/2408.02232)** · **[Sonar Foundation Agent](https://www.sonarsource.com/blog/introducing-sonar-foundation-agent/)**

</div>

---

## Highlights

- **51.6% resolve rate** on SWE-bench Verified (Jan 2025) at **$0.65 per issue** average cost
- **Sequential multi-agent pipeline** with immutable state transitions (functional paradigm)
- **Specification-driven patching** — infers function-level intended behavior to guide code generation
- **Reviewer agent** that cross-validates patches against reproducer tests and issue semantics
- **Self-correction loop** — diagnoses inapplicable patches, identifies the responsible agent, and replays from that stage
- **Interactive replay** — engineers inject feedback at any pipeline stage; the system replays from that point forward
- **8 programming languages** supported via tree-sitter AST-based code search
- **Provider-agnostic model layer** — OpenAI, Anthropic, Google, DeepSeek, AWS Bedrock, OpenRouter, Ollama
- **Acquired by Sonar** — evolved into the Sonar Foundation Agent ([#1 on SWE-bench, 79.2%, Feb 2026](https://www.sonarsource.com/company/press-releases/sonar-claims-top-spot-on-swe-bench-leaderboard/))

---

## Table of Contents

- [How It Works](#how-it-works)
- [System Architecture](#system-architecture)
  - [Pipeline Overview](#pipeline-overview)
  - [Orchestration Layer](#orchestration-layer)
  - [Action / State Machine](#action--state-machine)
  - [Agent Layer](#agent-layer)
  - [Specification Inference](#specification-inference)
  - [Multi-Language Code Search Backend](#multi-language-code-search-backend)
  - [Patch Generation & Validation](#patch-generation--validation)
  - [Self-Correction Loop](#self-correction-loop)
  - [Interactive Replay & Feedback](#interactive-replay--feedback)
  - [Model Abstraction Layer](#model-abstraction-layer)
- [Getting Started](#getting-started)
- [Usage](#usage)
- [Advanced Configuration](#advanced-configuration)
- [Project Structure](#project-structure)
- [Output Artifacts](#output-artifacts)
- [License](#license)

---

## How It Works

Given a bug report (GitHub issue, local file, or SWE-bench task), AutoCodeRover-Interactive autonomously:

1. **Reproduces** the bug — generates a test that triggers the reported failure
2. **Localizes** the fault — LLM-driven structural code search with optional Spectrum-Based Fault Localization (SBFL)
3. **Infers specifications** — extracts function-level intended behavior as natural-language specifications to guide patching
4. **Generates** a patch — dedicated patch-writing agent with iterative refinement
5. **Reviews** the patch — independent reviewer agent cross-validates against the reproducer test, issue statement, and inferred specifications
6. **Self-corrects** — if the patch is inapplicable, a self-fix agent diagnoses the root cause and replays the pipeline from the responsible stage
7. **Selects** the best patch — aggregates candidates across retries, runs regression validation, and selects via majority vote

The **Interactive** variant adds a **human-in-the-loop feedback mechanism**: engineers can inject corrections at the context retrieval or patch generation stages, and the system replays from that point with feedback incorporated into subsequent LLM prompts.

---

## System Architecture

### Pipeline Overview

The pipeline executes **sequentially**, not in parallel. Each stage produces an immutable state snapshot that the next stage extends.

```
                        ┌──────────────────────────┐
                        │          INPUT            │
                        │  GitHub Issue | Local File│
                        │  | SWE-bench Task         │
                        └────────────┬─────────────┘
                                     │
                                     ▼
                        ┌──────────────────────────┐
                        │   META-AGENT              │
                        │   (Orchestrator)           │
                        │                            │
                        │   Hardcoded: deterministic │
                        │   sequence of actions      │
                        │   LLM-driven: dynamic      │
                        │   action selection          │
                        └────────────┬─────────────┘
                                     │
              ┌──────────────────────┼── sequential ──────────────────┐
              │                      │                                │
              ▼                      ▼                                ▼
   ┌─────────────────┐   ┌─────────────────┐            ┌──────────────────┐
   │   Reproduce     │   │   SBFL          │            │ Context Retrieval│
   │   Bug           │──▶│   (optional)    │──────────▶ │ + Spec Inference │
   └─────────────────┘   └─────────────────┘            └────────┬─────────┘
                                                                  │
                                                                  ▼
                                                       ┌─────────────────┐
                                                       │   Write Patch   │
                                                       └────────┬────────┘
                                                                 │
                                                                 ▼
                                                       ┌─────────────────┐
                                                       │  Review Patch   │
                                                       └────────┬────────┘
                                                                 │
                                                    ┌────────────┴────────────┐
                                                    │                         │
                                                    ▼                         ▼
                                              patch OK?                 patch failed
                                                    │                         │
                                                    │                         ▼
                                                    │               ┌─────────────────┐
                                                    │               │   Self-Fix      │
                                                    │               │   Agent         │
                                                    │               └────────┬────────┘
                                                    │                        │
                                                    │              diagnose failure,
                                                    │              replay from the
                                                    │              responsible stage
                                                    │                        │
                                                    │                        └──────▶ (loops back to
                                                    │                                  Context Retrieval
                                                    │                                  or Write Patch)
                                                    │
                                                    ▼
                                      ┌──────────────────────────┐
                                      │    Select Patch          │
                                      │    (Best-of-N across     │
                                      │     retry iterations)    │
                                      └────────────┬─────────────┘
                                                   │
                                                   ▼
                                      ┌──────────────────────────┐
                                      │    selected_patch.json   │
                                      └──────────────────────────┘
```

**Key design decisions:**
- The pipeline is **sequential by default** (HardcodedMetaAgent). The LLM-driven MetaAgent variant can dynamically reorder actions but follows the same state contract.
- The Self-Fix Agent is **not a parallel peer** of Write Patch and Review Patch — it is a **recovery loop** triggered only on failure, routing corrective feedback back to the responsible upstream stage.
- Select Patch operates **across retry iterations**, not within a single pass. Multiple Write → Review → Self-Fix cycles produce candidate patches; Select Patch aggregates and picks the best one.

### Orchestration Layer

The system supports two orchestration strategies:

| Strategy | Class | How It Works |
|----------|-------|-------------|
| **Hardcoded** | `HardcodedMetaAgent` | Executes a deterministic sequence: Reproduce → SBFL → Context Retrieval → Write Patch → Review → Self-Fix. This is the default and most battle-tested mode. |
| **LLM-Driven** | `MetaAgent` | An LLM dynamically decides the next action based on accumulated state. Enabled with `--enable-meta-agent`. More flexible but less predictable. |

Both implement a **Python generator-based coroutine loop** — each action yields its type and receives a state-change description, enabling the orchestrator to make informed decisions about the next step.

### Action / State Machine

Every pipeline stage is modeled as an **Action** — an immutable, serializable unit of work. Actions consume and produce `ActionState`, a **frozen, slotted dataclass** that captures the full pipeline state. State transitions produce new `ActionState` instances (functional paradigm), ensuring clean reproducibility and enabling deterministic replay.

| Action | Purpose |
|--------|---------|
| `GenerateReproducer` | Generates a reproducing test for the bug via the `TestAgent`. |
| `GenerateSBFL` | Runs Spectrum-Based Fault Localization to rank suspicious code regions by coverage. |
| `ContextRetrieval` | Drives the `SearchAgent` through multi-round conversational code search, collecting buggy locations and function-level specifications. |
| `WritePatch` | Invokes the `PatchAgent` to generate a candidate patch from the search context and inferred specifications. Validates diff applicability. |
| `ReviewPatch` | Cross-validates the patch against the reproducer test using the `ReviewerAgent`. Produces a binary accept/reject decision with an explanation. |
| `SelfFix` | Diagnoses why a patch was inapplicable: collects failure reasons → analyzes root cause → identifies the responsible agent → generates corrective feedback → replays from that stage. |
| `SelectPatch` | Aggregates patches across retries, runs regression validation, and selects the best candidate via LLM-guided majority vote. |
| `TerminalExecution` | Executes shell commands via an LLM-driven terminal agent for exploratory debugging. |
| `FeedbackContext` | Re-runs context retrieval with human feedback injected into the LLM prompt. |
| `FeedbackPatch` | Re-runs patch generation with human feedback injected into the LLM prompt. |

Every action execution is **recorded to JSON** (`recording/*.json`), enabling full deterministic replay and interactive feedback injection at any point.

### Agent Layer

Each action delegates to specialized LLM agents:

| Agent | Role |
|-------|------|
| **SearchFrontend + SearchAgent** | Multi-round conversational code search. The LLM issues API-style calls (`search_class`, `search_method`, `search_code`, etc.) that are parsed by a `ProxyAgent` and dispatched to language-specific `SearchBackend` implementations. |
| **PatchAgent** | Generates patches from the search context and inferred specifications. Supports iterative retries with temperature escalation and validates diff applicability before committing. |
| **TestAgent** | Generates a reproducer test that triggers the reported bug. Validates reproduction by executing the test against the buggy program. Supports feedback-driven refinement. |
| **ReviewerAgent** | Executes the reproducer test against both the original and patched programs, then asks an LLM to judge correctness of **both** the patch and the test simultaneously. Produces structured accept/reject decisions with explanations. |
| **SelfFixAgent** | Analyzes inapplicable patches through a 4-step chain: collect failure reasons → analyze → identify the responsible upstream agent → generate targeted corrective feedback for replay. |
| **ProxyAgent** | Converts free-form LLM search responses into structured JSON API calls for the search backend. |
| **TerminalAgent** | Executes shell commands with timeout protection, captures stdout/stderr, and uses an LLM to review results and decide next steps. |
| **SelectionAgent** | Selects the best patch from multiple candidates by analyzing root causes and comparing candidate approaches against the issue description. |
| **TestBlacklistAgent** | Identifies irrelevant tests that should be excluded from regression validation. |

All agents use a **`MessageThread` abstraction** for conversation management, supporting serialization, template substitution, and tool-call interleaving.

### Specification Inference

A core contribution of the SpecRover extension is **iterative specification inference** — the system doesn't just find buggy code, it infers *why* the code should behave differently.

**Function summaries.** During context retrieval, whenever the `SearchAgent` retrieves a code snippet, it is explicitly prompted to analyze the *intended behavior* of that function in the context of the current issue. These function-level specifications serve as local guides for the `PatchAgent`, decomposing a repository-level issue into smaller, specification-driven function modification tasks.

**Reviewer feedback as meta-specification.** The `ReviewerAgent` produces structured feedback that reconciles the patch, the reproducer test, and the issue statement. This feedback is a reflection on the specifications inferred in previous steps — identifying where the inferred intent and the actual patch diverge. Critically, the reviewer examines both the patch and the test without assuming either is correct, enabling it to reject a faulty test while approving a correct patch.

**Evidence generation.** Along with the final patch, SpecRover outputs: buggy locations with their inferred intended behaviors, the reproducer test, the reviewer's rationale for acceptance/rejection, and the selection agent's reason for choosing the final patch. These artifacts serve as auditable evidence for why a particular fix was selected.

### Multi-Language Code Search Backend

The search backend uses **tree-sitter** parsers for language-aware structural code search, indexed at startup for fast lookup:

| Language | Supported Operations |
|----------|-----------------------------|
| Python | Classes, methods, functions, top-level definitions, code snippets |
| Java | Classes, methods, fields, imports |
| JavaScript | Functions, classes, methods, modules |
| TypeScript | Functions, classes, methods, modules |
| C / C++ | Functions, structs, headers |
| Go | Functions, types, interfaces |
| PHP | Classes, functions, methods |
| Text / Config | Plain-text fallback search for non-code files |

**Search APIs** available to the LLM agent: `search_class`, `search_method`, `search_method_in_class`, `search_method_in_file`, `search_class_in_file`, `search_code`, `search_code_in_file`, and more. Each API call is backed by indexed ASTs, not grep.

### Patch Generation & Validation

1. The `PatchAgent` prompts the LLM to produce **search/replace edit blocks** guided by the inferred specifications and buggy locations.
2. `patch_utils.py` parses these blocks and applies **fuzzy matching** against the original source — stripping line numbers, normalizing whitespace, and handling minor formatting differences.
3. A language-specific `PatchService` validates the result (e.g., Python AST lint check via `ast.parse` to catch syntax errors before committing).
4. Applied patches are registered in the immutable `PatchIndex` with their raw response, diff, and feedback history.
5. Validation runs the project's test suite (via Docker containers for SWE-bench tasks) to check for regressions.

### Self-Correction Loop

When the `ReviewerAgent` rejects a patch or the patch is inapplicable, the **Self-Fix Agent** activates:

1. **Collect** the specific failure reasons (inapplicable diff, test failure, reviewer rejection).
2. **Analyze** the failure through a Chain-of-Thought prompt that understands each agent's role in the pipeline.
3. **Identify** which upstream agent most likely caused the problem — typically the `Context Retrieval Agent` (wrong buggy locations) or the `Patch Agent` (referencing non-existent code elements).
4. **Generate** targeted corrective feedback for that agent.
5. **Replay** the pipeline from the identified stage using the same replay mechanism as interactive feedback.

This loop repeats up to a configurable retry limit. The most common recovery: the Self-Fix Agent detects that the Context Retrieval Agent proposed incorrect buggy locations (recognizable when the generated patch references code elements that don't exist at the proposed location), generates corrective feedback, and replays context retrieval.

### Interactive Replay & Feedback

The replay system is a **first-class architectural feature**, not an afterthought:

1. Every action execution is serialized to `recording/*.json`, capturing full state transitions — message threads, search results, patch diffs, review decisions.
2. An engineer reviews the output and identifies where the agent went wrong.
3. The `replay` subcommand loads the recorded session, replays the pipeline up to the feedback point (reloading saved intermediate states), then re-executes from that point with the feedback injected into subsequent LLM prompts.
4. Feedback can target **Context Retrieval** (e.g., "the search missed the relevant module") or **Patch Generation** (e.g., "the patch should modify `utils.py` instead").
5. Feedback is preserved in a `feedback_history` that accumulates across multiple rounds.

The key insight: **feedback at stage N implies satisfaction with stages 1 through N-1.** The system doesn't re-run the entire pipeline — it reloads saved states of preceding agents, re-executes only the targeted agent with feedback, and runs downstream agents on the new output. This makes interaction feel like a conversation rather than starting over.

### Model Abstraction Layer

The system provides a **provider-agnostic model layer** with auto-registration:

```
app/model/
├── common.py          # Abstract Model base, cost tracking, MODEL_HUB registry
├── register.py        # Reflection-based auto-discovery at startup
├── litellm_model.py   # Shared LiteLLM implementation
├── gpt.py             # OpenAI (GPT-4o, o1-mini, etc.)
├── claude.py          # Anthropic (Claude 3/3.5/4 Sonnet, Opus, Haiku)
├── deepseek.py        # DeepSeek V3 / R1
├── gemini.py          # Google Gemini (Vertex AI or API key)
├── bedrock.py         # AWS Bedrock
├── openrouter.py      # OpenRouter (multi-provider routing)
├── ollama.py          # Local models (Llama 3, etc.)
└── tools/             # Tool-use / function calling implementations
```

Design choices:
- **Singleton model registry** (`MODEL_HUB`) populated at startup via reflection-based auto-discovery — adding a new provider requires only a new file
- **Per-process cost accumulation** with configurable cost limits (`--cost-limit`)
- **Per-agent model assignment** via `--model-map` (e.g., Claude for search, GPT-4o for patching, a cheaper model for proxy parsing)
- **Automatic retry with exponential backoff** for transient API failures
- **Multimodal support** — image content in issue descriptions is converted to base64 data URIs

---

## Getting Started

### Prerequisites

- Python 3.11+
- Conda (Miniconda or Anaconda)
- Docker (required for SWE-bench mode)
- Git
- An API key for at least one supported LLM provider

### Installation — Docker (Recommended)

```bash
export OPENAI_API_KEY=sk-YOUR-KEY
# Or: export ANTHROPIC_API_KEY=sk-ant-...

docker build -f Dockerfile -t acr .
docker run -it -e OPENAI_API_KEY="${OPENAI_API_KEY}" acr
```

### Installation — Local (Conda)

```bash
git clone <repo-url> && cd acr-interactive

conda env create -f environment.yml
conda activate auto-code-rover

# Or via pip:
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
| `swe-bench` | Run on SWE-bench task instances for benchmarking and evaluation. |
| `scripted` | Run on a pre-configured project with metadata and bug report files. |
| `replay` | Replay a previous session with injected human feedback. |
| `extract-patches` | Post-process experiment outputs to extract all generated patches. |

### GitHub Issue Mode

```bash
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

### Local Issue Mode

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

```bash
# Single task
PYTHONPATH=. python app/main.py swe-bench \
  --model gpt-4o-2024-05-13 \
  --output-dir output \
  --task-id django__django-11133

# Batch execution
PYTHONPATH=. python app/main.py swe-bench \
  --model gpt-4o-2024-05-13 \
  --output-dir output \
  --task-list-file tasks.txt \
  --num-processes 4
```

> SWE-bench mode automatically builds Docker environment images for each task instance. See [EXPERIMENT.md](EXPERIMENT.md) for batch experiment configuration.

### Replay Mode (Interactive Feedback)

```bash
# Replay a previous session
PYTHONPATH=. python app/main.py replay <path-to-output-dir> \
  --model gpt-4o-2024-05-13 \
  --output-dir output

# Replay up to a specific action
PYTHONPATH=. python app/main.py replay <path-to-output-dir> \
  --replay-action 2 \
  --model gpt-4o-2024-05-13 \
  --output-dir output
```

The system reads `recording/*.json` files, detects feedback injections, and replays from the feedback point forward.

---

## Advanced Configuration

### Per-Agent Model Assignment

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

Format: `<agent-class>::<model-name>[::temperature]`

| Agent Class | Maps To |
|-------------|---------|
| `search` | Context retrieval / code search |
| `patch` | Patch generation |
| `reproducer` | Reproducer test generation |
| `proxy` | JSON proxy (search response parsing) |
| `review` | Patch reviewer |
| `select` | Patch selection |
| `terminal` | Shell command execution |
| `meta` | Meta-agent (LLM-driven orchestration) |
| `self_fix` | Self-fix diagnostic |
| `test_select` | Test blacklist |

### Config File for Batch Experiments

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

### Feature Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--model` | `gpt-3.5-turbo-0125` | Primary LLM model |
| `--model-temperature` | `0.0` | Default temperature for all agents |
| `--conv-round-limit` | `15` | Max conversation rounds for context retrieval |
| `--enable-sbfl` | `false` | Enable Spectrum-Based Fault Localization |
| `--enable-validation` | `false` | Validate patches against regression tests |
| `--enable-angelic` | `false` | Enable angelic debugging (experimental) |
| `--reproduce-and-review` | `false` | Enable the full reproduce → patch → review cycle |
| `--num-processes` | `1` | Parallel task processes |
| `--cost-limit` | `-1.0` (disabled) | Max API cost ($) before terminating |
| `--record` | `true` | Record actions for replay |

---

## Project Structure

```
acr-interactive/
├── app/
│   ├── main.py                  # Entry point — CLI parsing, task dispatch
│   ├── config.py                # Global configuration
│   ├── args.py                  # CLI argument definitions (ConfigArgParse)
│   ├── data.py                  # Core data models (Patch, Test, Review)
│   ├── run_task.py              # Task execution loop (retry, state management)
│   ├── raw_tasks.py             # Task types (GitHub, Local, SWE-bench, Scripted)
│   ├── postprocessing.py        # Patch extraction and SWE-bench formatting
│   ├── result_analysis.py       # Experiment result analysis and reporting
│   │
│   ├── actions/                 # Pipeline actions (state machine)
│   │   ├── action.py            # Abstract Action base class with record/replay
│   │   ├── context_retrieval.py # LLM-driven code search + spec inference
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
│   │       ├── feedback_history.py  # Feedback accumulation
│   │       ├── feedback_index.py    # Agent-stage enum for feedback routing
│   │       ├── patch_selection.py   # Final selection result
│   │       └── reproduction_info.py # Reproducer execution results
│   │
│   ├── agents/                  # LLM agent implementations
│   │   ├── agent_search.py      # Search frontend (multi-round LLM conversation)
│   │   ├── agent_write_patch.py # Patch generation agent
│   │   ├── agent_reproducer.py  # Bug reproducer test agent
│   │   ├── agent_meta.py        # Meta-agent / hardcoded orchestrator
│   │   ├── agent_proxy.py       # JSON proxy (LLM → structured API calls)
│   │   ├── agent_command.py     # Terminal command execution agent
│   │   ├── agent_self_fix.py    # Self-fix diagnostic agent
│   │   ├── agent_select.py      # Patch selection agent
│   │   ├── agent_test_blacklist.py  # Test relevance filtering
│   │   ├── reviewer.py          # Patch + test review agent
│   │   ├── message_thread.py    # Conversation thread abstraction
│   │   ├── prompts/             # 28 prompt templates (.md files)
│   │   ├── search/              # Code search backend
│   │   │   ├── search_backend.py    # Search service registry & dispatch
│   │   │   ├── search_manage.py     # Search orchestration
│   │   │   ├── search_service.py    # Abstract search service interface
│   │   │   ├── python/          # Python AST search (tree-sitter)
│   │   │   ├── java/            # Java search
│   │   │   ├── js/              # JavaScript search
│   │   │   ├── ts/              # TypeScript search
│   │   │   ├── cpp/             # C/C++ search
│   │   │   ├── go/              # Go search
│   │   │   ├── php/             # PHP search
│   │   │   └── text/            # Plain-text fallback
│   │   └── testing/             # Test execution and result types
│   │
│   ├── model/                   # LLM provider abstraction layer
│   │   ├── common.py            # Abstract Model, cost tracking, MODEL_HUB
│   │   ├── register.py          # Auto-discovery model registration
│   │   ├── litellm_model.py     # LiteLLM shared implementation
│   │   ├── gpt.py               # OpenAI
│   │   ├── claude.py            # Anthropic
│   │   ├── deepseek.py          # DeepSeek
│   │   ├── gemini.py            # Google Gemini
│   │   ├── bedrock.py           # AWS Bedrock
│   │   ├── openrouter.py        # OpenRouter
│   │   ├── ollama.py            # Local Ollama
│   │   └── tools/               # Tool-use implementations
│   │
│   ├── patch/                   # Patch parsing, application, validation
│   │   ├── patch_utils.py       # Diff extraction, fuzzy matching, application
│   │   ├── patch_service.py     # Abstract patch service interface
│   │   └── python/              # Python-specific validation (AST lint)
│   │
│   ├── sbfl/                    # Spectrum-Based Fault Localization
│   │   ├── sbfl.py              # SBFL orchestration
│   │   ├── swe_bench.py         # SWE-bench SBFL integration
│   │   └── utils.py             # Coverage data collation
│   │
│   ├── tasks/                   # Task abstractions
│   │   ├── abstract_task.py     # Abstract Task (setup, validation, cleanup)
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

After a run, the output directory contains:

| File | Description |
|------|-------------|
| `selected_patch.json` | Final selected patch and selection rationale |
| `meta.json` | Task metadata, model configuration, model map |
| `cost.json` | Total API cost, input/output token counts, wall-clock time |
| `output_0/` | First retry iteration |
| `output_0/conv_patch_*.json` | Full LLM conversation threads for patch generation |
| `output_0/patch_raw_*.md` | Raw LLM patch responses |
| `output_0/extracted_patch_*.diff` | Extracted unified diffs |
| `output_0/extract_status.json` | Patch extraction result (applicable / unmatched / etc.) |
| `output_0/search_0/` | Context retrieval results and tool call layers |
| `output_0/recording/*.json` | Serialized action states for replay |
| `output_0/display.jsonl` | Structured display log for UI rendering |
| `output_0/bug_locs.dill` | Serialized bug location objects |

---

## License

This project is **dual-licensed**:

- **Non-commercial use** (including academic research): [GNU Affero General Public License v3](LICENSE)
- **Commercial use**: Contact the authors for a commercial license.

---

## References

- **AutoCodeRover**: Y. Zhang, H. Ruan, Z. Fan, and A. Roychoudhury. "AutoCodeRover: Autonomous Program Improvement." ACM ISSTA 2024. [DOI](https://doi.org/10.1145/3650212.3680384)
- **SpecRover**: H. Ruan, Y. Zhang, and A. Roychoudhury. "SpecRover: Code Intent Extraction via LLMs." [arXiv:2408.02232](https://arxiv.org/abs/2408.02232)
- **Sonar Foundation Agent**: [Introducing the Sonar Foundation Agent](https://www.sonarsource.com/blog/introducing-sonar-foundation-agent/) (Feb 2026) · [#1 on SWE-bench](https://www.sonarsource.com/company/press-releases/sonar-claims-top-spot-on-swe-bench-leaderboard/) (79.2% Verified, 52.62% Full)

---

## Contact

For issues, questions, or collaboration inquiries:

- 📧 Email: [info@autocoderover.dev](mailto:info@autocoderover.dev)
- 💬 Discord: [AutoCodeRover Community](https://discord.com/invite/ScXsdE49JY)
