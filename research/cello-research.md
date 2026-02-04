# CELLO: Code Evaluation LLM Leaderboard Open
## Comprehensive Research Document

**Author:** Prepared for Chris (MOFA/AOSF)
**Date:** February 3, 2026
**Version:** 1.0

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Part 1: SonarSource LLM Leaderboard Analysis](#2-part-1-sonarsource-llm-leaderboard-analysis)
3. [Part 2: Reference Site Analysis (CELLO Concept)](#3-part-2-reference-site-analysis)
4. [Part 3: Open Source LLM Code Evaluation Landscape](#4-part-3-open-source-llm-code-evaluation-landscape)
5. [Part 4: CELLO Solution Proposal](#5-part-4-cello-solution-proposal)
6. [Part 5: Code Quality Metrics Framework](#6-part-5-code-quality-metrics-framework)
7. [Appendices](#7-appendices)

---

## 1. Executive Summary

The evaluation of LLM-generated code quality is a critical and rapidly evolving field. SonarSource's "Coding Personalities of Leading LLMs" leaderboard represents the current commercial state-of-the-art, measuring code quality, security, and maintainability across ~4,442 programming tasks. However, it has significant limitations: it predominantly evaluates closed-source models, uses a limited test corpus, relies on proprietary tooling (SonarQube), and evaluates primarily synthetic benchmarks rather than real-world codebases.

This document proposes **CELLO (Code Evaluation LLM Leaderboard Open)** — a fully open source, community-driven alternative that addresses these gaps by:
- Testing **both** open source and closed source LLMs
- Using **10,000+** test cases derived from real open source projects
- Employing **100% OSI-approved open source tooling** for evaluation
- Providing **transparent, reproducible methodology**
- Including real-world projects (makepad-d3, D3.js, matplotlib, makepad-matplot) as evaluation targets

---

## 2. Part 1: SonarSource LLM Leaderboard Analysis

### 2.1 What It Is

SonarSource publishes "The Coding Personalities of Leading LLMs" — an LLM code quality leaderboard available at:
- **Main Report:** https://www.sonarsource.com/the-coding-personalities-of-leading-llms/
- **Interactive Leaderboard:** https://www.sonarsource.com/the-coding-personalities-of-leading-llms/leaderboard/

The leaderboard is part of SonarSource's "State of Code" research series and is described as going "beyond standard benchmarks" to evaluate how LLMs write production-quality code.

### 2.2 What It Measures

The SonarSource leaderboard evaluates LLM-generated code across five key dimensions:

#### Verbosity Measurement
- **Lines of Code (LOC):** Total lines generated across all tasks, including blank lines and comments
- **Token Count:** Language-agnostic measure of code volume and content density
- **Code Density:** Ratio of executable statements to total lines

#### Complexity Measurement
- **Cyclomatic Complexity:** Measures the number of independent execution paths
- Per-model complexity profiling

#### Communication & Documentation
- **Comment Density:** Percentage of code that is comments (e.g., Claude 3.7 Sonnet achieves 16.4% comment density)
- Documentation quality assessment

#### Software Quality Analysis (SonarQube)
- **Reliability:** Bugs that affect software capability and operational effectiveness
- **Security:** Vulnerabilities (exploitable code weaknesses) and Security Hotspots (security-sensitive code requiring manual review)
- **Maintainability:** Code smells indicating design weaknesses, technical debt, increased risk of bugs
- **Issue Density:** Issues per thousand lines of code (Issues/kLOC)
- **Severity Classification:** BLOCKER, CRITICAL, MAJOR, MINOR, INFO

#### Functional Performance
- **Pass Rate:** Percentage of benchmark tests passed (e.g., Opus 4.5 Thinking: 83.62%, Claude Sonnet 4: 77.04%)

### 2.3 Models Tested (as of February 2026)

The leaderboard has expanded from 5 initial models to **16 models**, including:

| Model | Organization | Type | Pass Rate | Issue Density |
|-------|-------------|------|-----------|---------------|
| Opus 4.5 Thinking | Anthropic | Closed | 83.62% | 15.15 Issues/kLOC |
| GPT-5.2 High | OpenAI | Closed | ~81% | ~26+ Issues/kLOC |
| Gemini 3 Pro | Google | Closed | ~81% | Lower |
| Claude Sonnet 4 | Anthropic | Closed | 77.04% | — |
| Claude 3.7 Sonnet | Anthropic | Closed | 72.46% | — |
| GPT-4o | OpenAI | Closed | — | 26.08 Issues/kLOC |
| GPT-5-minimal | OpenAI | Closed | — | 26.65 Issues/kLOC |
| Llama 3.2 90B | Meta | Open-weights | 61.47% | >26 Issues/kLOC |
| OpenCoder-8B | Community | Open | Lower | 32.45 Issues/kLOC |

**Key observation:** Only **2 out of 16 models** (Llama 3.2 and OpenCoder-8B) are open source/open-weights. The remaining 14 are closed-source commercial models.

### 2.4 Methodology Details

- **Task Count:** 4,442 identical programming tasks performed by each LLM
- **Analysis Tool:** SonarQube (proprietary static analysis)
- **Language Focus:** Primarily Java (based on available blog posts), with limited multi-language coverage
- **Evaluation Type:** Synthetic benchmarks — generated code solutions to predefined problems

### 2.5 Key Findings from SonarSource

1. **"Newer" ≠ "Better Code":** GPT-5-minimal (Aug '25) has higher issue density (26.65) than GPT-4o (May '24) at 26.08
2. **Verbosity Disparity:** GPT-5.2 High generates ~974K LOC vs. Gemini 3 Pro's ~289K LOC for the same tasks — a 3.4x difference
3. **Security Blindspots Universal:** All models demonstrate fundamental lack of security awareness
4. **Upgrades Increase Risk:** Claude Sonnet 4 is 93% more likely to produce BLOCKER vulnerabilities than its predecessor
5. **Technical Debt Dominant:** ~X% of all issues create long-term technical debt
6. **Llama 3.2 90B Security Failure:** 70.73% of its vulnerabilities are BLOCKER severity

### 2.6 Challenges and Limitations

#### 🔴 Critical Limitation 1: Closed-Source Model Bias
- **14 of 16 models tested are closed-source** (GPT-4, GPT-5, Claude, Gemini variants)
- Only 2 open source models: Llama 3.2 90B and OpenCoder-8B
- **Missing entirely:** DeepSeek Coder V2/V3, StarCoder2, Qwen2.5-Coder, CodeLlama, Codestral, Phi-3/4, Yi-Coder, Mistral-based code models
- This gives a fundamentally skewed view of the open source LLM ecosystem

#### 🔴 Critical Limitation 2: Limited Test Corpus
- Only **4,442 test cases** — while substantial, may not be comprehensive enough for production-level evaluation
- Compare to:
  - BigCodeBench: 1,140 tasks (but more practically complex)
  - SWE-bench: 2,294 real GitHub issues
  - HumanEval+: 164 tasks (but with rigorous test cases)
  - LiveCodeBench: 300+ tasks (contamination-free, continuously updated)
- 4,442 synthetic tasks cannot capture the full spectrum of real-world software engineering challenges

#### 🔴 Critical Limitation 3: Proprietary Tooling
- Uses **SonarQube** as the sole evaluation engine — a proprietary tool
- Results cannot be independently reproduced without a SonarQube license
- No open source alternative tooling offered for verification
- Creates vendor lock-in in the evaluation methodology itself

#### 🟡 Limitation 4: Language Coverage Gaps
- Blog content focuses heavily on **Java**
- Unclear coverage of:
  - **Rust** (increasingly important for systems programming)
  - **TypeScript/JavaScript** (web ecosystem)
  - **Python** (ML/data science)
  - **Go** (cloud infrastructure)
  - **C/C++** (embedded/systems)
- No evidence of multi-language comparative analysis

#### 🟡 Limitation 5: Synthetic Benchmarks Only
- Tests use **predefined programming tasks**, not real-world codebases
- Does not evaluate:
  - Code integration into existing projects
  - Refactoring capabilities
  - Understanding of project-specific patterns and conventions
  - Performance in context of real dependencies
  - Handling of edge cases from production systems

#### 🟡 Limitation 6: No Contamination Controls
- No mechanisms to detect if models have been trained on the benchmark tasks
- No temporal splitting (unlike LiveCodeBench)
- Results may be inflated for models that have seen similar patterns in training

#### 🟡 Limitation 7: Commercial Motivation
- SonarSource has a commercial interest in promoting SonarQube as the solution
- The report naturally concludes with "use SonarQube to verify AI code"
- Independent, vendor-neutral evaluation is needed

#### 🟡 Limitation 8: Static Snapshot
- The leaderboard is updated periodically, not continuously
- By the time results are published, newer model versions may already exist
- No community contribution mechanism for testing additional models

---

## 3. Part 2: Reference Site Analysis

### 3.1 CELLO Concept Overview

**Source:** https://csheargm.github.io/aosf/cello.html
**GitHub:** https://github.com/agentic-osf/cello (referenced but repository not yet public)

The CELLO (Code Evaluation LLM Leaderboard Open) concept as presented on the reference site proposes:

### 3.2 Core Principles
- **100% Open Source:** All evaluation tools, methodology, and results are fully transparent
- **OSI-Approved Tools Only:** No proprietary dependencies
- **Open Source Models Focus:** While not excluding closed models, prioritizes open source LLM evaluation

### 3.3 Metrics Framework (from reference site)

| Metric Category | Weight | Description |
|----------------|--------|-------------|
| Code Quality | 20% | Linting, style, idiomatic code |
| Security Analysis | 20% | Vulnerability detection, secure coding patterns |
| Performance | 15% | Runtime efficiency, resource usage |
| Test Coverage | 20% | Generated test quality and coverage |
| Documentation | 10% | Comments, docstrings, README quality |
| Compilation Success | 15% | Whether code compiles/runs without errors |

### 3.4 Tools Used
- **Semgrep** — Security analysis (open source SAST)
- **Clippy** — Rust linting
- **ESLint** — JavaScript linting
- **Bandit** — Python security analysis
- **Custom benchmarks** — Project-specific evaluation harnesses
- **Test harnesses** — Automated test execution

### 3.5 Test Projects (Translation-Based Evaluation)
The reference site proposes a unique approach: evaluating LLMs by having them **translate** well-known C projects to Rust:
- **Redis → Rust**
- **SQLite → Rust**
- **Git → Rust**
- **Nginx → Rust**
- **PostgreSQL Extensions**
- **Custom benchmarks**

This is a brilliant approach because:
1. The source projects are well-understood and well-tested
2. Translation tasks test deep understanding of both source and target languages
3. Results can be verified against the original project's test suites
4. It simulates real-world code migration tasks

### 3.6 Platform Statistics (as reported)
- **15+** Models Evaluated
- **127** Total Evaluations
- **5** Languages Supported
- **89.2%** Average Success Rate

### 3.7 Key Differentiators from SonarSource
1. **Fully transparent methodology** — available on GitHub
2. **Open source tools only** — reproducible by anyone
3. **Translation-based evaluation** — more realistic than synthetic tasks
4. **Community-driven** — open to contributions

---

## 4. Part 3: Open Source LLM Code Evaluation Landscape

### 4.1 Existing Benchmarks & Leaderboards

#### 4.1.1 HumanEval+ / EvalPlus
- **Source:** https://evalplus.github.io/leaderboard.html
- **Tasks:** 164 (HumanEval) + 399 (MBPP+)
- **Methodology:** Function-level code generation with rigorous test suites
- **Strengths:** Well-established baseline, open source, extensively used
- **Limitations:** Small task count, only Python, primarily tests correctness not quality
- **Notable:** Includes both open and closed models, uses pass@1 with greedy decoding

#### 4.1.2 BigCodeBench
- **Source:** https://bigcode-bench.github.io/
- **Tasks:** 1,140 (Full Set) / ~150 (Hard Set)
- **Methodology:** Practical, challenging programming tasks requiring library usage
- **Modes:** Complete (code completion) and Instruct (NL instructions)
- **Strengths:** More practical than HumanEval, tests real library usage
- **Limitations:** Still synthetic tasks, primarily correctness-focused
- **Notable:** Distinguishes open data (💚) from open weights (💙), tracks contamination potential

#### 4.1.3 LiveCodeBench
- **Source:** https://livecodebench.github.io/
- **Paper:** arXiv:2403.07974 (UC Berkeley, MIT, Cornell)
- **Tasks:** 300+ continuously updated coding problems from LeetCode, AtCoder, Codeforces
- **Key Innovation:** **Contamination-free** — problems annotated with release dates, allowing temporal splitting
- **Evaluates:** Code generation, self-repair, test output prediction, code execution
- **Findings:**
  - DeepSeek models show stark performance drops on post-training-cutoff problems (contamination signal)
  - Closed API models generally outperform open models
  - HumanEval performance may be inflated due to overfitting
  - Open fine-tuned models cluster in an "overfit" region on HumanEval vs. LiveCodeBench plots

#### 4.1.4 SWE-bench
- **Source:** https://www.swebench.com/
- **Tasks:** 2,294 (Full) / 500 (Verified) / 300 (Lite) / 517 (Multimodal)
- **Methodology:** Real GitHub issues — models must understand codebases and generate patches
- **Strengths:** Most realistic benchmark — actual software engineering work
- **Limitations:** Complex evaluation setup, high cost per evaluation
- **Variants:** SWE-bench Bash Only (standardized environment), SWE-bench Multimodal (visual elements)
- **Notable:** mini-SWE-agent achieves 65% on Verified with just 100 lines of Python

#### 4.1.5 Aider LLM Leaderboard
- **Source:** https://aider.chat/docs/leaderboards/
- **Methodology:** Tests LLMs in an actual coding assistant context (aider tool)
- **Key Metrics:** Edit success rate, cost per task
- **Top Performers (as of Feb 2026):**
  - GPT-5 (high): 88.0% — $29.08
  - GPT-5 (medium): 86.7% — $17.69
  - o3-pro (high): 84.9% — $146.32
  - Gemini 2.5 Pro: 83.1% — $49.88
  - DeepSeek-V3.2-Exp (Reasoner): 74.2% — $1.30
  - DeepSeek R1 (0528): 71.4% — $4.80
- **Strengths:** Tests real-world coding assistant use case, includes cost data
- **Open source model highlights:**
  - Qwen3 235B: 59.6%
  - DeepSeek V3 (0324): 55.1% — $1.12
  - Qwen2.5-Coder-32B: 16.4%

### 4.2 What None of Them Measure

**Critical gap:** None of the existing benchmarks comprehensively evaluate **code quality** in the SonarSource sense:

| Benchmark | Correctness | Security | Code Smells | Complexity | Maintainability | Test Coverage | Real Projects |
|-----------|------------|----------|-------------|------------|-----------------|---------------|---------------|
| HumanEval+ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| BigCodeBench | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| LiveCodeBench | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| SWE-bench | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Aider | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| SonarSource | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| **CELLO (proposed)** | **✅** | **✅** | **✅** | **✅** | **✅** | **✅** | **✅** |

### 4.3 Open Source LLMs for Code — Current Performance Landscape

#### Tier 1: Strong Performers
| Model | Organization | Params | Open Weights | Open Data | Key Strengths |
|-------|-------------|--------|--------------|-----------|---------------|
| DeepSeek-V3.2-Exp | DeepSeek | ~685B MoE | ✅ | ❌ | 74.2% on Aider, $1.30/task |
| DeepSeek R1 (0528) | DeepSeek | ~685B MoE | ✅ | ❌ | Strong reasoning, 71.4% on Aider |
| Qwen3-235B-A22B | Alibaba | 235B MoE | ✅ | ✅ | 59.6% on Aider, open data |
| Kimi K2 | Moonshot AI | — | ✅ | ❌ | 59.1% on Aider, $1.24/task |

#### Tier 2: Cost-Effective Performers
| Model | Organization | Params | Aider Score | Cost |
|-------|-------------|--------|------------|------|
| DeepSeek V3 (0324) | DeepSeek | 685B MoE | 55.1% | $1.12 |
| DeepSeek Chat V3 (prev) | DeepSeek | 685B MoE | 48.4% | $0.34 |
| Qwen3-32B | Alibaba | 32B | 40.0% | $0.76 |

#### Tier 3: Smaller Models
| Model | Organization | Params | Notes |
|-------|-------------|--------|-------|
| Qwen2.5-Coder-32B | Alibaba | 32B | 16.4% on Aider (diff mode), strong HumanEval |
| OpenHands-LM-32B | All Hands AI | 32B | Purpose-built for SWE tasks |
| Codestral 25.01 | Mistral | — | 11.1%, specialized for code |
| Gemma 3 27B | Google | 27B | 4.9% on Aider |

### 4.4 Key Insights for CELLO

1. **Cost vs. Quality Trade-off:** DeepSeek models offer 10-100x cost advantage over GPT-5/Claude while achieving 60-70% of their performance
2. **Open source is catching up fast:** DeepSeek-V3.2 at 74.2% vs. GPT-5 at 88% — the gap is narrowing
3. **Size matters less than architecture:** Qwen3-235B (MoE) outperforms many dense models of similar parameter count
4. **No benchmark tests code quality:** The biggest gap in the ecosystem

---

## 5. Part 4: CELLO Solution Proposal

### 5.1 Vision Statement

**CELLO (Code Evaluation LLM Leaderboard Open)** is an open source platform for comprehensive evaluation of LLM code generation quality. Unlike existing benchmarks that focus solely on functional correctness, CELLO evaluates the full spectrum of production-readiness: security, maintainability, complexity, test coverage, and documentation — using 100% open source tools against real-world codebases.

### 5.2 Core Design Principles

1. **Fully Open Source:** All tools, methodology, scoring algorithms, and results are OSI-approved open source
2. **Inclusive:** Tests both open source AND closed source LLMs — no model is excluded
3. **Real-World Focus:** Evaluates against actual open source projects, not just synthetic benchmarks
4. **Comprehensive:** Tests 10,000+ cases across multiple languages and project types
5. **Transparent:** Every score is reproducible; every methodology decision is documented
6. **Community-Driven:** Anyone can submit models, propose test cases, or contribute tooling
7. **Continuously Updated:** New problems and models added regularly; contamination controls built in

### 5.3 Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    CELLO Platform                            │
├─────────────┬─────────────┬──────────────┬─────────────────┤
│  Task Engine │  LLM Runner │  Analysis    │  Leaderboard    │
│              │             │  Pipeline    │  Dashboard      │
│  • Task DB   │  • API      │  • Semgrep   │  • Web UI       │
│  • Real repos│  • Local    │  • Clippy    │  • API          │
│  • Generators│  • Ollama   │  • ESLint    │  • Embeds       │
│              │             │  • Bandit    │  • Comparisons  │
│              │             │  • Ruff      │                 │
│              │             │  • SonarQube*│                 │
│              │             │  • Custom    │                 │
└─────────────┴─────────────┴──────────────┴─────────────────┘
         │                          │                │
    ┌────┴────┐              ┌──────┴──────┐   ┌────┴────┐
    │  Task   │              │  Scoring    │   │  Results │
    │  Types  │              │  Engine     │   │  Store   │
    │         │              │             │   │         │
    │ • Synth │              │ • Weighted  │   │ • JSON  │
    │ • Trans │              │ • Normalized│   │ • DB    │
    │ • Real  │              │ • Comparable│   │ • API   │
    │ • Fix   │              │             │   │         │
    └─────────┘              └─────────────┘   └─────────┘
```

*SonarQube included as optional comparison layer, not a required dependency

### 5.4 Task Categories

CELLO proposes **four distinct task categories**, each testing different aspects of LLM code capability:

#### Category 1: Synthetic Tasks (~3,000 tasks)
Similar to existing benchmarks but with quality-focused evaluation:
- Function implementation tasks with quality scoring
- Algorithm challenges with complexity analysis
- API design tasks with maintainability assessment
- Security-focused tasks (write secure auth, crypto, input validation)

#### Category 2: Translation Tasks (~2,000 tasks)
Inspired by the CELLO reference site's approach:
- **JavaScript → Rust:** D3.js module translations
- **Python → Rust:** matplotlib component translations
- **C → Rust:** Systems software translations
- **Cross-language API preservation:** Maintain semantics across languages

#### Category 3: Real-World Project Tasks (~4,000 tasks)
Using actual open source projects as evaluation targets:
- Bug fix tasks from real issue trackers
- Feature implementation from real feature requests
- Refactoring tasks from real code review feedback
- Test generation for existing untested code

#### Category 4: Integration Tasks (~1,000+ tasks)
- Add features to existing codebases
- Resolve merge conflicts
- Update dependencies
- Write documentation for existing code

**Total: 10,000+ test cases** (2.25x the SonarSource corpus)

### 5.5 Example Project Evaluation Targets

#### 5.5.1 makepad-d3 (https://github.com/mofa-org/makepad-d3)
- **Language:** Rust
- **Description:** D3.js-compatible data visualization library for Makepad's GPU-accelerated rendering
- **Complexity:** High — 40+ chart types, scales, layouts, geographic projections, force-directed graphs
- **Modules:** data, scale, axis, shape (7 curve interpolations), color (4 color spaces), layout (force, hierarchy), geo (4 projections), interaction (zoom, brush, tooltip), component
- **Why ideal for CELLO:**
  - Rich type system usage (Rust generics, traits)
  - Complex algorithms (force simulation, Sankey, treemap)
  - GPU rendering integration
  - Mathematical precision requirements
  - Well-structured module system for isolated evaluation
- **Evaluation tasks:**
  - Implement a new chart type (e.g., "implement a Funnel chart")
  - Add a new scale type (e.g., Diverging scale)
  - Write tests for existing modules
  - Fix reported bugs in layout algorithms
  - Translate specific D3.js modules to Rust

#### 5.5.2 D3.js (https://github.com/d3/d3)
- **Language:** JavaScript
- **Description:** The foundational data visualization library; 112K+ GitHub stars, 4,510 commits
- **Complexity:** Very high — modular architecture with 30+ sub-packages
- **Why ideal for CELLO:**
  - Extremely well-tested and documented
  - Complex mathematical algorithms
  - Extensive API surface
  - Active maintenance for 10+ years
  - Well-known patterns that test LLM understanding
- **Evaluation tasks:**
  - Implement missing features in D3 modules
  - Write comprehensive tests for edge cases
  - Optimize performance of specific algorithms
  - Document complex interpolation functions
  - Translate D3 modules to other languages

#### 5.5.3 matplotlib (https://github.com/matplotlib/matplotlib)
- **Language:** Python (with C extensions)
- **Description:** Comprehensive library for static, animated, and interactive visualizations
- **Complexity:** Very high — publication-quality output, multi-backend support
- **Why ideal for CELLO:**
  - Mature, production-grade Python codebase
  - Mix of Python and C for performance-critical paths
  - Extensive test suite (CI with codecov)
  - NumPy integration and scientific computing patterns
  - Complex object hierarchy and inheritance
- **Evaluation tasks:**
  - Fix real bugs from issue tracker
  - Add new plot types
  - Write tests for uncovered code paths
  - Optimize rendering performance
  - Improve documentation for complex APIs

#### 5.5.4 makepad-matplot (https://github.com/mofa-org/makepad-matplot)
- **Language:** Rust
- **Description:** Matplotlib-compatible plotting library for Makepad
- **Complexity:** High — translation of Python matplotlib concepts to Rust
- **Why ideal for CELLO:**
  - Direct comparison with matplotlib (Python → Rust translation quality)
  - Tests understanding of both Python and Rust idioms
  - GPU-accelerated rendering integration
  - Real-world code translation evaluation
- **Evaluation tasks:**
  - Implement matplotlib features not yet ported
  - Write idiomatic Rust equivalents of Python patterns
  - Generate test suites matching matplotlib's test coverage
  - Evaluate API design decisions

### 5.6 LLMs to Evaluate

CELLO will comprehensively test models across the entire spectrum:

#### Closed Source Models
| Model | Organization | Reasoning | Priority |
|-------|-------------|-----------|----------|
| GPT-5 / GPT-5.2 | OpenAI | Yes | High |
| GPT-4o / GPT-4.1 | OpenAI | No | High |
| o3 / o4-mini | OpenAI | Yes | High |
| Claude Opus 4.5 | Anthropic | Yes | High |
| Claude Sonnet 4 | Anthropic | Yes | High |
| Claude 3.7 Sonnet | Anthropic | Optional | Medium |
| Gemini 2.5 Pro | Google | Yes | High |
| Gemini 2.5 Flash | Google | Yes | Medium |
| Grok 3 / Grok 4 | xAI | Yes | Medium |

#### Open Source Models (Critical — underrepresented in SonarSource)
| Model | Organization | Params | Priority |
|-------|-------------|--------|----------|
| DeepSeek-V3.2 | DeepSeek | 685B MoE | **Critical** |
| DeepSeek R1 | DeepSeek | 685B MoE | **Critical** |
| DeepSeek Coder V2 | DeepSeek | 236B MoE | **Critical** |
| Qwen3-235B | Alibaba | 235B MoE | **Critical** |
| Qwen2.5-Coder-32B | Alibaba | 32B | **Critical** |
| Qwen2.5-Coder-7B | Alibaba | 7B | High |
| CodeLlama-70B | Meta | 70B | High |
| Llama 3.3 70B | Meta | 70B | High |
| StarCoder2-15B | BigCode | 15B | High |
| Codestral | Mistral | — | High |
| Phi-4 | Microsoft | 14B | Medium |
| Yi-Coder-9B | 01.AI | 9B | Medium |
| OpenCoder-8B | Community | 8B | Medium |
| Granite-Code-34B | IBM | 34B | Medium |
| OpenHands-LM-32B | All Hands | 32B | Medium |
| Kimi K2 | Moonshot | — | Medium |

### 5.7 Open Source Tooling Stack

All evaluation tools must be OSI-approved open source:

| Tool | Purpose | License | Language Coverage |
|------|---------|---------|-------------------|
| **Semgrep** | SAST / Security Analysis | LGPL-2.1 | Python, JS, Go, Java, Ruby, Rust, C, etc. |
| **Clippy** | Rust Linting | MIT/Apache-2.0 | Rust |
| **ESLint** | JavaScript/TypeScript Linting | MIT | JS/TS |
| **Bandit** | Python Security Analysis | Apache-2.0 | Python |
| **Ruff** | Python Linting (fast) | MIT | Python |
| **Pylint** | Python Code Quality | GPL-2.0 | Python |
| **mypy** | Python Type Checking | MIT | Python |
| **cargo-audit** | Rust Dependency Audit | MIT/Apache-2.0 | Rust |
| **cargo-clippy** | Rust Code Quality | MIT/Apache-2.0 | Rust |
| **rust-analyzer** | Rust Analysis | MIT/Apache-2.0 | Rust |
| **PMD** | Java/Apex Code Analysis | BSD | Java |
| **SpotBugs** | Java Bug Detection | LGPL | Java |
| **Checkstyle** | Java Style Analysis | LGPL-2.1 | Java |
| **golangci-lint** | Go Linting | GPL-3.0 | Go |
| **cppcheck** | C/C++ Analysis | GPL-3.0 | C/C++ |
| **radon** | Python Complexity | MIT | Python |
| **lizard** | Cyclomatic Complexity | MIT | 20+ languages |
| **tokei** | Lines of Code Counter | MIT/Apache-2.0 | 150+ languages |
| **cargo-tarpaulin** | Rust Code Coverage | MIT/Apache-2.0 | Rust |
| **pytest-cov** | Python Code Coverage | MIT | Python |
| **istanbul/nyc** | JS Code Coverage | ISC | JS/TS |

### 5.8 Interactive Leaderboard Features

The CELLO web dashboard will provide:

1. **Overall Rankings** — Weighted composite scores across all dimensions
2. **Per-Dimension Rankings** — Sort by security, quality, performance, etc.
3. **Per-Language Rankings** — How models perform in Rust vs. Python vs. JavaScript
4. **Per-Task-Type Rankings** — Synthetic vs. translation vs. real-world
5. **Model Comparison** — Head-to-head comparison of any two models
6. **Trend Analysis** — How model performance changes over time
7. **Cost-Efficiency View** — Quality per dollar (for API-based models)
8. **Open Source Spotlight** — Dedicated view for open source model comparison
9. **Detailed Reports** — Per-model breakdown with example code snippets
10. **API Access** — Programmatic access to all leaderboard data
11. **Community Submissions** — Submit your own model evaluations
12. **Embed Widgets** — Embeddable leaderboard widgets for other sites

### 5.9 Technical Implementation

#### Repository Structure
```
cello/
├── README.md
├── LICENSE (Apache-2.0)
├── CONTRIBUTING.md
├── docs/
│   ├── methodology.md
│   ├── scoring.md
│   ├── adding-models.md
│   └── adding-tasks.md
├── tasks/
│   ├── synthetic/
│   ├── translation/
│   ├── real-world/
│   └── integration/
├── runners/
│   ├── api_runner.py        # For API-based models
│   ├── local_runner.py      # For local models (Ollama, vLLM)
│   └── config/
├── analyzers/
│   ├── security.py          # Semgrep, Bandit integration
│   ├── quality.py           # Lint tool integration
│   ├── complexity.py        # Cyclomatic complexity analysis
│   ├── coverage.py          # Test coverage measurement
│   ├── documentation.py     # Comment density, docstring analysis
│   └── compilation.py       # Build success verification
├── scoring/
│   ├── weights.py
│   ├── normalization.py
│   └── ranking.py
├── dashboard/
│   ├── frontend/            # React/Next.js leaderboard UI
│   ├── api/                 # REST API for results
│   └── embeds/              # Embeddable widgets
├── results/
│   ├── raw/                 # Raw analysis output
│   ├── scored/              # Scored results
│   └── leaderboard.json     # Current leaderboard data
└── tests/
```

#### Execution Pipeline
```
1. Task Selection → Select task set for evaluation
2. LLM Execution → Generate code from each model
3. Compilation → Verify code compiles/runs
4. Test Execution → Run provided test suites
5. Static Analysis → Run all open source analyzers
6. Metric Extraction → Extract quality metrics
7. Scoring → Apply weighted scoring algorithm
8. Ranking → Generate leaderboard rankings
9. Publication → Update dashboard and API
```

---

## 6. Part 5: Code Quality Metrics Framework

### 6.1 Metric Categories & Definitions

#### 6.1.1 Functional Correctness (Weight: 15%)

| Metric | Description | Tool | Scale |
|--------|-------------|------|-------|
| Compilation Rate | % of tasks that compile successfully | Language compiler | 0-100% |
| Test Pass Rate | % of provided tests passed | pytest, cargo test, jest | 0-100% |
| Functional Equivalence | Output matches expected behavior | Custom harness | 0-100% |

#### 6.1.2 Security (Weight: 20%)

| Metric | Description | Tool | Scale |
|--------|-------------|------|-------|
| Vulnerability Count | Number of vulnerabilities found | Semgrep | Count |
| Vulnerability Density | Vulnerabilities per kLOC | Semgrep | Ratio |
| Severity Distribution | % Blocker/Critical/Major/Minor | Semgrep | Distribution |
| Injection Resistance | Resistance to injection patterns | Custom rules | 0-100% |
| Crypto Safety | Use of secure cryptographic patterns | Semgrep, Bandit | 0-100% |
| Auth Patterns | Secure authentication code | Custom rules | 0-100% |

#### 6.1.3 Code Quality / Smells (Weight: 20%)

| Metric | Description | Tool | Scale |
|--------|-------------|------|-------|
| Code Smell Count | Total code smells detected | ESLint, Clippy, Ruff, PMD | Count |
| Code Smell Density | Smells per kLOC | Multiple | Ratio |
| Dead Code | Unused variables, functions, imports | Language-specific linters | Count |
| Duplication | Copy-pasted or duplicated code blocks | Custom / jscpd | % |
| Naming Quality | Adherence to naming conventions | Linters | Score |
| Idiomatic Usage | Use of language-specific idioms | Language linters | Score |

#### 6.1.4 Complexity (Weight: 15%)

| Metric | Description | Tool | Scale |
|--------|-------------|------|-------|
| Cyclomatic Complexity | Number of independent execution paths | lizard, radon | Number |
| Cognitive Complexity | Human-perceived complexity | Custom (Sonar-inspired) | Number |
| Nesting Depth | Maximum nesting depth | Custom | Number |
| Function Length | Average/max lines per function | tokei + custom | Lines |
| Verbosity Ratio | LOC per task vs. median | tokei | Ratio |

#### 6.1.5 Maintainability (Weight: 15%)

| Metric | Description | Tool | Scale |
|--------|-------------|------|-------|
| Technical Debt Ratio | Estimated remediation time vs. dev time | Custom | Ratio |
| Modularity | Appropriate use of functions/modules | Custom | Score |
| Dependency Hygiene | Appropriate dependency usage | Audit tools | Score |
| Error Handling | Proper error handling patterns | Linters + custom | Score |
| Type Safety | Use of type annotations/generics | mypy, TypeScript, Rust compiler | Score |

#### 6.1.6 Test Quality (Weight: 10%)

| Metric | Description | Tool | Scale |
|--------|-------------|------|-------|
| Test Coverage | Line/branch coverage of generated tests | tarpaulin, pytest-cov, istanbul | 0-100% |
| Test Meaningfulness | Tests actually verify behavior (not trivial) | Custom analysis | Score |
| Edge Case Coverage | Coverage of boundary conditions | Custom analysis | Score |
| Assertion Density | Assertions per test function | Custom | Ratio |

#### 6.1.7 Documentation (Weight: 5%)

| Metric | Description | Tool | Scale |
|--------|-------------|------|-------|
| Comment Density | % of lines that are comments | tokei | % |
| Docstring Coverage | % of public functions with docstrings | pydocstyle, rustdoc | 0-100% |
| README Quality | Quality of generated documentation | Custom NLP analysis | Score |
| Inline Explanation | Explanatory comments for complex logic | Custom | Score |

### 6.2 Evaluation Methodology Using Example Repos

#### 6.2.1 makepad-d3 Evaluation Protocol

**Task Types Generated from makepad-d3:**

| Task Category | Count | Example |
|--------------|-------|---------|
| New feature implementation | 40 | "Implement a Radar chart in the chart zoo" |
| Bug fix (seeded) | 30 | "Fix the force simulation convergence bug" |
| Module translation (JS→Rust) | 50 | "Translate d3-scale-chromatic to Rust" |
| Test generation | 30 | "Write comprehensive tests for LinearScale" |
| Refactoring | 20 | "Refactor GeoPath to use trait objects" |
| Documentation | 15 | "Document the Sankey layout algorithm" |
| Performance optimization | 15 | "Optimize force simulation for 10K nodes" |
| **Total** | **200** | |

**Evaluation Process:**
1. Clone makepad-d3 at a fixed commit hash
2. Present task + relevant source files to each LLM
3. Collect generated code
4. Run `cargo build` (compilation check)
5. Run `cargo test` (test pass check)
6. Run `cargo clippy` (quality check)
7. Run Semgrep with Rust rules (security check)
8. Run lizard for complexity analysis
9. Run tokei for verbosity metrics
10. Run cargo-tarpaulin for coverage (test generation tasks)
11. Score and aggregate

#### 6.2.2 D3.js Evaluation Protocol

**Task Types Generated from D3:**

| Task Category | Count | Example |
|--------------|-------|---------|
| Module implementation | 50 | "Implement d3-contour from scratch" |
| Bug fix (from real issues) | 40 | "Fix issue #XXXX in d3-scale" |
| Translation (JS→Rust) | 60 | "Translate d3-interpolate to Rust" |
| Test generation | 30 | "Write tests for d3-hierarchy treemap" |
| API design | 20 | "Design a streaming data API for d3-array" |
| **Total** | **200** | |

**Evaluation Process:**
1. Clone d3 monorepo at fixed commit
2. Present task + relevant d3 module source
3. Collect generated code
4. Run `npm test` or `node --check` (compilation/syntax)
5. Run ESLint (quality)
6. Run Semgrep with JS rules (security)
7. Run jest with coverage (test tasks)
8. Score and aggregate

#### 6.2.3 matplotlib Evaluation Protocol

**Task Types Generated from matplotlib:**

| Task Category | Count | Example |
|--------------|-------|---------|
| Bug fix (from real issues) | 60 | "Fix issue #XXXX in axes.py" |
| Feature implementation | 40 | "Add 3D surface plot interpolation" |
| Translation (Python→Rust) | 40 | "Translate colormap module to Rust" |
| Test generation | 30 | "Write tests for backend_agg" |
| Refactoring | 20 | "Refactor Artist hierarchy to reduce coupling" |
| Documentation | 10 | "Document the transform pipeline" |
| **Total** | **200** | |

#### 6.2.4 makepad-matplot Evaluation Protocol

**Task Types:**

| Task Category | Count | Example |
|--------------|-------|---------|
| Feature parity implementation | 50 | "Implement contour plot matching matplotlib" |
| API translation (Python→Rust) | 40 | "Translate matplotlib.pyplot API to Rust" |
| Test generation | 30 | "Write tests for color mapping" |
| Performance optimization | 20 | "Optimize GPU rendering for scatter plots" |
| Integration tasks | 10 | "Integrate makepad-matplot with makepad-d3 scales" |
| **Total** | **150** | |

#### 6.2.5 Additional Synthetic & Cross-Project Tasks

| Category | Count |
|----------|-------|
| Pure algorithmic tasks (multi-language) | 2,000 |
| Security-focused tasks | 500 |
| Concurrency/async tasks | 300 |
| Error handling tasks | 200 |
| Cross-project integration | 100 |
| Systems programming tasks | 400 |
| Web development tasks | 500 |
| Data processing tasks | 500 |
| **Subtotal** | **4,500** |

**Grand Total: ~5,250 project-derived + ~4,500 synthetic = ~9,750 tasks**
(Easily expandable to 10,000+ with community contributions)

### 6.3 Scoring and Ranking System

#### 6.3.1 Per-Task Scoring

Each task receives a score from 0-100 across each metric category:

```
Task Score = Σ (Category Weight × Category Score)

Where:
  Functional Correctness: 15% weight
  Security:               20% weight
  Code Quality:           20% weight
  Complexity:             15% weight
  Maintainability:        15% weight
  Test Quality:           10% weight
  Documentation:           5% weight
```

#### 6.3.2 Category Scoring Details

**Security Score (0-100):**
```
Security = 100 - (Blocker_vulns × 20) - (Critical_vulns × 10) - (Major_vulns × 5) - (Minor_vulns × 2)
Clamped to [0, 100]
```

**Code Quality Score (0-100):**
```
Quality = 100 - (smell_density × normalization_factor)
Where normalization_factor adjusts for language-specific baseline smell rates
```

**Complexity Score (0-100):**
```
Complexity = 100 × (1 - (actual_complexity - optimal_complexity) / max_complexity)
Where optimal_complexity is derived from reference implementations
```

#### 6.3.3 Model-Level Aggregation

```
Model Score = Mean(all task scores)

With additional reporting:
  - Median score (robust to outliers)
  - P25/P75 scores (consistency measure)
  - Per-language breakdown
  - Per-task-type breakdown
  - Cost efficiency = Model Score / Cost per 1000 tasks
```

#### 6.3.4 Ranking Tiers

| Tier | Score Range | Label |
|------|------------|-------|
| S | 90-100 | Production-Ready |
| A | 80-89 | High Quality |
| B | 70-79 | Good with Review |
| C | 60-69 | Needs Significant Review |
| D | 50-59 | Use with Caution |
| F | <50 | Not Recommended for Production |

#### 6.3.5 Specialized Rankings

In addition to the overall leaderboard, CELLO provides:

1. **Security Champion:** Ranked by security score only
2. **Quality King:** Ranked by code quality + maintainability
3. **Speed Demon:** Ranked by functional correctness + compilation rate
4. **Documentation Hero:** Ranked by documentation quality
5. **Best Value:** Ranked by overall score / cost
6. **Open Source GOAT:** Best open source model overall
7. **Small Model Star:** Best model under 15B parameters
8. **Language Specialist:** Best model per programming language

---

## 7. Appendices

### Appendix A: Comparison Matrix — CELLO vs. Existing Solutions

| Feature | SonarSource | EvalPlus | BigCodeBench | SWE-bench | LiveCodeBench | Aider | CELLO |
|---------|------------|---------|--------------|-----------|---------------|-------|-------|
| Open source | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Tests open source LLMs | 🟡 (2/16) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Tests closed LLMs | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Security analysis | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Code quality metrics | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Complexity analysis | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Test coverage eval | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Real-world projects | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ |
| Contamination control | ❌ | ❌ | 🟡 | 🟡 | ✅ | ❌ | ✅ |
| Cost tracking | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| Community contributions | ❌ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| Multi-language | 🟡 | ❌ (Python) | ❌ (Python) | ❌ (Python) | ❌ (Python) | ✅ | ✅ |
| Test cases | 4,442 | 563 | 1,140 | 2,294 | 300+ | 225 | 10,000+ |
| Reproducible | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

### Appendix B: Open Source Tool Licenses

| Tool | License | OSI Approved |
|------|---------|-------------|
| Semgrep | LGPL-2.1 | ✅ |
| ESLint | MIT | ✅ |
| Clippy | MIT/Apache-2.0 | ✅ |
| Bandit | Apache-2.0 | ✅ |
| Ruff | MIT | ✅ |
| Pylint | GPL-2.0 | ✅ |
| PMD | BSD-4-Clause | ✅ |
| SpotBugs | LGPL-2.1 | ✅ |
| lizard | MIT | ✅ |
| tokei | MIT/Apache-2.0 | ✅ |

### Appendix C: Sources & References

1. **SonarSource "Coding Personalities of Leading LLMs"**
   - Main report: https://www.sonarsource.com/the-coding-personalities-of-leading-llms/
   - Leaderboard: https://www.sonarsource.com/the-coding-personalities-of-leading-llms/leaderboard/
   - Java analysis blog: https://www.sonarsource.com/blog/how-to-choose-your-llm-without-ruining-your-java-code/

2. **CELLO Reference Implementation**
   - Site: https://csheargm.github.io/aosf/cello.html
   - GitHub: https://github.com/agentic-osf/cello

3. **BigCodeBench**
   - Site: https://bigcode-bench.github.io/
   - Paper: https://openreview.net/forum?id=YrycTjllL0
   - HuggingFace: https://huggingface.co/spaces/bigcode/bigcodebench-leaderboard

4. **EvalPlus / HumanEval+**
   - Site: https://evalplus.github.io/leaderboard.html
   - Paper: https://openreview.net/forum?id=1qvx610Cu7

5. **LiveCodeBench**
   - Site: https://livecodebench.github.io/
   - Paper: arXiv:2403.07974

6. **SWE-bench**
   - Site: https://www.swebench.com/
   - Variants: Full (2,294), Verified (500), Lite (300), Multimodal (517)

7. **Aider LLM Leaderboard**
   - https://aider.chat/docs/leaderboards/

8. **Example Repositories**
   - makepad-d3: https://github.com/mofa-org/makepad-d3
   - D3.js: https://github.com/d3/d3
   - matplotlib: https://github.com/matplotlib/matplotlib
   - makepad-matplot: https://github.com/mofa-org/makepad-matplot

### Appendix D: Roadmap

| Phase | Timeline | Deliverables |
|-------|----------|-------------|
| Phase 1: Foundation | Months 1-2 | Task framework, 2,000 synthetic tasks, 3 analyzers, CLI tool |
| Phase 2: Real-World Tasks | Months 2-3 | 800 tasks from 4 example repos, translation pipeline |
| Phase 3: First Leaderboard | Month 3 | Evaluate 10+ models, publish initial leaderboard |
| Phase 4: Dashboard | Months 3-4 | Interactive web dashboard, API, embed widgets |
| Phase 5: Community | Months 4-6 | Open contributions, model submission pipeline, CI/CD |
| Phase 6: Scale | Months 6-12 | 10,000+ tasks, 30+ models, additional languages |

---

*Document prepared February 3, 2026. For questions or contributions, contact the CELLO project team.*
*Repository: https://github.com/agentic-osf/cello*
