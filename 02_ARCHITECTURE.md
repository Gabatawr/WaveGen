# System Blueprint: 02 - Component Architecture

This document details the individual components of the WaveGen system and their responsibilities. The architecture is designed to be modular and extensible.

### 1. The `WaveController`
The central orchestrator and the main facade of the system. It owns all other components and manages the main generation loop. Its primary responsibility is to sequence the operations: receiving the prompt, calling the `GuardrailEvaluator`, invoking the `TemplateMatcher`, executing the dynamic path (metrics -> policy -> action), and caching the final result.

### 2. `BaseLLMBackend` (Integration Layer)
An abstract base class that defines a standardized interface for interacting with any LLM. This is the system's only direct link to a model.
-   **Responsibilities**:
    -   Tokenizing and detokenizing text.
    -   Executing a single generation step to produce `logits`.
    -   **Performing logit calibration** (e.g., temperature scaling) to ensure metrics are comparable across different models.
    -   **Exporting provenance data** used to generate the `ProvenanceHash` for the current configuration.
-   **Implementations**: `HuggingFaceBackend`. Others (like `vLLM`, `Llama.cpp`) can be added without changing the core system.

### 3. `Buffer`
A simple, efficient, fixed-size queue (a `collections.deque`) that holds the most recent `StepObservations`. It acts as the short-term, working memory for the system, providing the data window for analysis.

### 4. `MetricsCalculator`
A stateless component that takes a window of `StepObservations` from the `Buffer` and computes a dictionary of raw metrics, which are then aggregated into a single `Coherence Coefficient (Γ)`.
-   **Key Input Metrics**:
    -   `entropy`: The Shannon entropy of the logit distribution.
    -   `js_divergence`: The "jitter" between consecutive logit distributions.
    -   `top_k_jaccard`: The stability of the top-K most likely tokens between steps.
    -   `semantic_drift`: The cosine distance between embeddings of top-K tokens.
-   **Primary Output**: `Coherence Coefficient (Γ)`, a single value representing the stability of the trajectory.

### 5. `EmaSmoother`
A stateful component that applies an Exponential Moving Average to the `Coherence Coefficient (Γ)`. This is crucial for preventing spurious, short-lived fluctuations from triggering a policy action prematurely. It ensures that decisions are based on stable trends, not noise.

### 6. `PolicyEngine`
The "brain" of the system. It is a stateless component that contains the core decision-making logic.
-   **Inputs**: The smoothed `Coherence Coefficient (Γ)` and a `PolicyConfig` object.
-   **`PolicyConfig`**: A configuration object containing the specific numerical thresholds that trigger different actions. It implements **hysteresis** (separate `commit` and `release` thresholds) to prevent rapid state-switching.
-   **Logic**: Before committing to a `JUMP`, it can perform a **"probe-decode"**: generating a few speculative tokens to validate that the jump is appropriate.
-   **Output**: A single `PolicyAction` (`JUMP`, `STOP`, etc.).

### 7. `TrajectoryCache`
The long-term memory of the system. Each cached entry maintains counters for `uses`, `successes`, and `rollbacks`.
-   **Responsibilities**:
    -   `create_signature()`: Generates a `TrajectorySignature` (including `ProvenanceHash`) from a window of observations.
    -   `update()`: Saves a `TrajectorySignature` and its corresponding `GeneratedBlock`.
    -   `lookup()`: Takes a signature, performs a fast FAISS search, and then validates the result with SimHash and `ProvenanceHash` checks.
    -   `record_verdict()`: Updates the `successes` or `rollbacks` counter for a cached entry after a `Guardrail` decision.
    -   **Eviction Policy**: Implements a **weighted LRU policy**. The "value" of an entry is a function of its recency, usage frequency, and success rate (`successes / uses`). Entries with high rollback counts are prioritized for eviction.

### 8. `TemplateMatcher`
The "reflex" system. It maintains a library of common boilerplate patterns. It performs a fast match on the *current text* (potentially using `minihash` instead of regex for performance) and, if a match is found, returns a `GeneratedBlock` to trigger a `SHORTCUT` action.

### 9. `GuardrailEvaluator`
A critical safety component that validates a `GeneratedBlock` *before* it is inserted into the generation context.
-   **Responsibilities**:
    -   Performs a cascade of checks based on the content type (e.g., code vs. text).
    -   **For Code**: Syntax checks (`ast.parse`), static analysis (`ruff`), safety checks (disallowed imports/functions), and optional sandboxed execution of micro-tests using technologies like **WASM or gVisor**.
    -   **For Text**: Regex invariants, semantic similarity checks, and topic/policy filters.
-   **Outcome**: If any check fails, the block is rejected, and the `WaveController` falls back to standard generation. The failure is logged via `TrajectoryCache.record_verdict()`.

### 10. System Configuration & Privacy
The system's behavior is controlled via a central configuration that defines:
-   **Policy Profiles**: Domain-specific thresholds for the `PolicyEngine` (e.g., for `code`, `sql`, `text`).
-   **Privacy Levels**: A setting that controls what is stored in the cache to meet different security requirements.
    -   `private` (default): Stores full data, including raw text, enabling all `Guardrail` checks.
    -   `team_shared`: Stores token IDs and hashes, but no raw text. Disables `Guardrail` checks that require full content.
