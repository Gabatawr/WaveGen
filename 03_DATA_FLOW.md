# System Blueprint: 03 - Data Flow & Interaction Lifecycle

This document describes the end-to-end lifecycle of a single `wave_controller.generate()` call, illustrating how the components interact.

### Initial State
-   The `WaveController` and all its sub-components are initialized.
-   The `TrajectoryCache` has loaded any existing data from disk.
-   The `Buffer` and `EmaSmoother` are empty.

### Lifecycle Steps

1.  **[User] -> [WaveController]**: A user calls `generate(prompt, max_tokens)`.

2.  **[WaveController] -> [Backend]**: The `prompt` is tokenized into `generated_token_ids`.

3.  **[WaveController] -> [TemplateMatcher] (Shortcut Path)**:
    -   The current text is passed to the `TemplateMatcher`.
    -   **If** a template is matched:
        -   `[WaveController] -> [GuardrailEvaluator]`: The corresponding `GeneratedBlock` is retrieved and sent for validation.
        -   **If** the block passes validation:
            -   Its tokens are appended to `generated_token_ids`.
            -   The generation loop **terminates**. The final text is returned.
        -   **Else (Rollback)**: The `SHORTCUT` is aborted, the failure is logged, and the loop continues to the dynamic path.

4.  **[WaveController] -> [Backend] (Generation Step)**:
    -   The current `generated_token_ids` are sent to the `Backend`.
    -   The `Backend` returns the `logits` for the next token after applying model-specific calibration.
    -   A `token_id` is sampled from the `logits`.

5.  **[WaveController] -> [Buffer] (Observation)**:
    -   A `StepObservation` is created and added to the `Buffer`.

6.  **[WaveController] -> [MetricsCalculator] -> [EmaSmoother] (Analysis)**:
    -   The window of observations from the `Buffer` is used to compute the raw `Coherence Coefficient (Γ)`.
    -   The `EmaSmoother` returns the smoothed `Γ`.

7.  **[WaveController] -> [PolicyEngine] (Decision)**:
    -   The smoothed `Γ` is passed to the `PolicyEngine`.
    -   The `PolicyEngine` evaluates its rules (including hysteresis) and returns a single `PolicyAction`.

8.  **[WaveController] (Action Execution)**:
    -   **If `CONTINUE`**: The `token_id` from step 4 is appended to `generated_token_ids`. The loop repeats from step 3.
    -   **If `STOP`**: The generation loop **terminates**.
    -   **If `JUMP`**:
        -   `[WaveController] -> [TrajectoryCache]` (Create Signature): A `TrajectorySignature` is created from the current `Buffer` window.
        -   `[TrajectoryCache]` (Lookup & Validate): The signature is used to find a candidate block via FAISS, then validated with SimHash and `ProvenanceHash`.
        -   **If** a valid candidate is found:
            -   `[WaveController] -> [GuardrailEvaluator]`: The candidate `GeneratedBlock` is sent for validation.
            -   **If** the block passes Guardrail checks:
                -   `[TrajectoryCache] -> record_verdict(success)`: The success is logged for the cached entry.
                -   (Optional) A "probe-decode" may run a few speculative steps. If the probe is successful, the block's tokens are appended, and the loop **terminates**.
                -   If the probe fails, the `JUMP` is aborted, the failure is logged, and the loop continues.
            -   **Else (Rollback)**: The `JUMP` is aborted. `[TrajectoryCache] -> record_verdict(failure)` is called to log the failure and lower the entry's trust score. The loop continues.
        -   **Else** (no valid match in cache): The `JUMP` action is ignored, and the loop continues.

9.  **[WaveController] (Finalization)**:
    -   The loop terminates due to `max_tokens`, `STOP`, `SHORTCUT`, or a successful `JUMP`.
    -   **If** the termination was "natural" (i.e., not interrupted by a jump or shortcut):
        -   `[WaveController] -> [TrajectoryCache]` (Update): A final signature is created from the initial part of the generation, and the rest of the generated content is packaged into a `GeneratedBlock`. This new pair is saved to the cache with its counters initialized.
    -   `[WaveController] -> [Backend]`: The final `generated_token_ids` are detokenized.
    -   **[WaveController] -> [User]**: The final text is returned.
