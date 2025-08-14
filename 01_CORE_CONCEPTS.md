# System Blueprint: 01 - Core Concepts

This document defines the key abstract nouns of the WaveGen system.

### Trajectory
A **Trajectory** is the fundamental unit of analysis. It is a time-ordered sequence of `StepObservations` that represents the complete history of a generation process. It is more than just the generated text; it is a record of the LLM's internal state at each step.

### Step Observation
A **Step Observation** is an atomic snapshot of the LLM's state at a single generation step. It contains:
-   `token_id`: The specific token that was chosen.
-   `logits`: The full probability distribution over the entire vocabulary for that step. This is the raw "thought process" of the model, from which all our metrics are derived.
-   `step_index`: The position of the observation in the sequence.

### Coherence Coefficient (Γ)
A derived, composite metric that summarizes the stability of the generation process over a window of recent steps. It is calculated by the `MetricsCalculator` from several underlying metrics (e.g., entropy, Jaccard similarity of top-K tokens, JS-divergence between steps) and is the primary input for the `PolicyEngine`. A high Γ indicates a stable, predictable pattern, making the trajectory a candidate for a `JUMP`.

### Trajectory Signature
A **Trajectory Signature** is a compressed, high-dimensional "fingerprint" of a Trajectory. It is designed to be a unique but "fuzzy" identifier, allowing us to find similar, but not necessarily identical, trajectories. It has three components:
1.  **Semantic Synopsis (FAISS Vector):** An averaged vector of the token embeddings in the trajectory. This captures the *conceptual meaning* or the "what" of the trajectory. It is used for fast, approximate nearest-neighbor search.
2.  **Rank Pattern Hash (SimHash):** A hash derived from the sequence of token IDs. This captures the *syntactic structure* or the "rhythm" of the trajectory. It is used as a high-precision filter after a candidate is found via the semantic search.
3.  **Provenance Hash:** A cryptographic hash (e.g., BLAKE3) of the generation's metadata. This ensures that caches are strictly isolated between different models and configurations, preventing "cross-contamination." The metadata includes: `model_id`, `tokenizer_id`, `quantization_type`, `sampling_algorithm`, and binned values for `temperature` and `top_p`.

### Generated Block
A **Generated Block** is a self-contained, sequential chunk of content (represented as both text and token IDs) that can be inserted into an ongoing generation. It is the "payload" of a `JUMP` or `SHORTCUT` action. Each block is associated with the `TrajectorySignature` that leads to it.

### Policy Action
A **Policy Action** is the discrete decision made by the `PolicyEngine` at each step. The primary actions are:
-   `CONTINUE`: Proceed with normal, token-by-token generation.
-   `JUMP`: A cache-based acceleration. Abort token-by-token generation and insert a `GeneratedBlock` from the `TrajectoryCache`.
-   `SHORTCUT`: A template-based acceleration. Abort generation and insert a predefined `GeneratedBlock` from the `TemplateMatcher`.
-   `STOP`: Terminate the generation early if the system detects a stable and complete output, even if `max_tokens` has not been reached.
-   `THINK`: A research-oriented action triggered by high uncertainty. Pauses the main generation to invoke a secondary, more complex reasoning process (e.g., Chain-of-Thought) to resolve ambiguity.
