# System Blueprint: 04 - Development Roadmap

This document outlines a strategic, multi-phase plan for developing the WaveGen system from its current prototype state into a production-ready, robust, and valuable tool.

## Phase 1: Quantitative Validation

**Goal:** To empirically prove that the system provides a tangible performance benefit and to understand its characteristics.
**Status:** In Progress.

-   [ ] **1.1: Establish Baseline Performance:**
    -   Develop a `benchmark_runner.py` script.
    -   Primary Metrics: **Speedup Ratio**, **Token Savings Ratio**.

-   [ ] **1.2: Measure Component Contribution (Ablation Study):**
    -   Extend the benchmark runner to allow for ablation studies (e.g., `JUMP` only, `SHORTCUT` only).

-   [ ] **1.3: Implement Advanced Quality Assurance Metrics:**
    -   Add domain-specific quality metrics to the benchmark.
    -   **For Text:** ROUGE, BLEU, semantic similarity.
    -   **For Code:** **Functional equivalence** (e.g., by running unit tests against the generated code) and **AST-based similarity** to ensure structural correctness.

## Phase 2: Policy Tuning & Intelligence

**Goal:** To maximize performance gains by optimizing the system's decision-making process.

-   [ ] **2.1: Tune `PolicyConfig` Thresholds:**
    -   Systematically experiment with policy thresholds to find a robust default configuration.

-   [ ] **2.2: Develop Domain-Specific Policy Profiles:**
    -   Create different `PolicyConfig` profiles tailored for `code`, `sql`, and `text`.

-   [ ] **2.3: Implement Logit Calibration:**
    -   Implement and test logit calibration strategies (e.g., temperature scaling) within the `BaseLLMBackend` to ensure consistent policy behavior across different models.

-   [ ] **2.4: (Research) Advanced `THINK` Action:**
    -   Prototype a `THINK` action for when the model is highly uncertain (low `Γ`).

## Phase 3: Production Hardening & Safety

**Goal:** To make the system reliable and safe for use in production environments.

-   [ ] **3.1: Implement Comprehensive Guardrail System:**
    -   Develop the `GuardrailEvaluator` with a full suite of checks, including sandboxed execution for code.

-   [ ] **3.2: Implement Failure Telemetry:**
    -   Log detailed context whenever a `GeneratedBlock` is rejected by a guardrail to enable debugging and auto-tuning.

-   [ ] **3.3: Finalize Cache Eviction Strategy:**
    -   Implement and test the full weighted LRU eviction policy (`uses`, `successes`, `rollbacks`) in the `TrajectoryCache`.

## Phase 4: Usability & Observability

**Goal:** To package the system into a clean, usable, and transparent library.

-   [ ] **4.1: Comprehensive Unit Testing:**
    -   Build out a full `tests/` suite.

-   [ ] **4.2: Refine API and CLI:**
    -   Create a clean, user-friendly public API and a `Typer`-based CLI.

-   [ ] **4.3: Develop Visualization & Monitoring Tools:**
    -   Create visualization scripts for debugging and integrate Prometheus exporters for real-time monitoring.

-   [ ] **4.4: Implement Cache Warm-up Strategy:**
    -   Develop a **"shadow-warming"** mode where a new cache can be populated in the background without affecting user-facing latency, solving the "cold start" problem for new projects or models.

-   [ ] **4.5: Finalize Documentation:**
    -   Review and finalize all public docstrings and blueprint documents.

## Phase 5: Advanced Cache Intelligence

**Goal:** To evolve the cache from a simple storage mechanism into a self-optimizing knowledge base.

-   [ ] **5.1: Implement Proactive Cache Optimization (`CacheGardener`):**
    -   Develop a background process that periodically analyzes the cache.
    -   **Consolidation:** Finds and merges similar trajectories into a single, more robust entry.
    -   **Abstraction:** Generalizes cached entries by replacing specific literals with placeholders (e.g., using AST-normalization for code to abstract variable names).

-   [ ] **5.2: Develop Offline Cache Priming Tool (`CacheWarmer`):**
    -   Create a standalone utility that can pre-populate a WaveGen cache by analyzing a large, existing corpus of data (e.g., a Git repository or a document store). This allows for day-one performance on new projects.
