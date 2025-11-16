# System Blueprint: 05 - DynamicLoRA Integration

This document describes how WaveGen integrates with the DynamicLoRA hierarchical memory system.

## Overview: Two Complementary Systems

**WaveGen** optimizes **inference** (how fast the model generates).

**DynamicLoRA** optimizes **learning** (how the model adapts over time).

Together, they form a complete memory system:

```
┌──────────────────────────────────────────────┐
│  WaveGen (Inference Space)                   │
│  - Working cache (L1/L2)                     │
│  - Trajectory-based (sequences of tokens)    │
│  - External to model parameters              │
│  - Temporal scale: seconds to minutes        │
└─────────────┬────────────────────────────────┘
              │
              │ Cache hit rate signals
              │ Circuit extraction
              │ Provenance coordination
              ↓
┌──────────────────────────────────────────────┐
│  DynamicLoRA (Parameter Space)               │
│  - Long-term memory (RAM/disk)               │
│  - Circuit-based (activation patterns)       │
│  - Internal model weights                    │
│  - Temporal scale: minutes to months         │
└──────────────────────────────────────────────┘
```

---

## Integration Points

### 1. **Provenance Hash Coordination**

WaveGen cache keys must include LoRA state to ensure correctness.

**Before LoRA integration:**
```python
cache_key = hash(prompt_prefix)
```

**With LoRA integration:**
```python
cache_key = hash(
    prompt_prefix +
    lora_snapshot.provenance_hash  # includes all active LoRA layers
)
```

**Why:**
- Same prompt with different LoRA states → different outputs
- Cache lookup must be LoRA-aware

**Implementation:**
```python
class TrajectoryCache:
    def create_signature(self, trajectory, lora_snapshot):
        semantic_vector = self.compute_semantic_synopsis(trajectory)

        rank_pattern = self.compute_rank_pattern_hash(trajectory)

        # NEW: Include LoRA provenance
        provenance_hash = hash(
            model_id=self.backend.model_id,
            tokenizer_id=self.backend.tokenizer_id,
            lora_snapshot_hash=lora_snapshot.hash(),  # ← includes all layers
            sampling_params=self.sampling_config
        )

        return TrajectorySignature(
            semantic=semantic_vector,
            syntactic=rank_pattern,
            provenance=provenance_hash
        )
```

---

### 2. **Cache Hit Rate → Migration Signal**

WaveGen tracks how often each cached trajectory is used. This becomes a signal for LoRA consolidation.

**High hit rate = frequently used circuit → should migrate to LoRA**

**Flow:**
```python
# WaveGen observes generation
trajectory = observe_generation(prompt, lora_snapshot)

if coherence(trajectory) > threshold:
    # Cache it
    signature = cache.create_signature(trajectory, lora_snapshot)
    cache.store(signature, generated_block)

# Later: cache is used
if hit := cache.lookup(signature):
    cache.record_hit(signature)

# Periodically: check hit rates
for entry in cache.entries:
    if entry.hit_rate > consolidation_threshold:
        # Signal to LoRA: internalize this pattern
        circuit = extract_circuit(entry.trajectory)
        lora_hierarchy.request_consolidation(circuit)

        # Evict from cache (no longer needed)
        cache.evict(entry)
```

**Parameters:**
- `consolidation_threshold`: Hit rate triggering LoRA migration (e.g., 0.7 = 70% of requests)
- Varies by tier:
  - Context → Agent: 0.5 (50% hit rate)
  - Agent → Swarm: 0.7 (70% hit rate)
  - Swarm → Domain: 0.85 (85% hit rate)

---

### 3. **Circuit Extraction from Trajectories**

WaveGen trajectories contain activation history. DynamicLoRA extracts circuits from this.

**WaveGen trajectory:**
```python
Trajectory = [
    StepObservation(
        token_id=128,
        logits=[...],
        step_index=0
    ),
    ...
]
```

**Extended for LoRA:**
```python
StepObservation(
    token_id=128,
    logits=[...],
    activations={  # NEW: layer activations
        'layer_3': tensor([...]),
        'layer_7': tensor([...]),
        ...
    },
    step_index=0
)
```

**Circuit extraction:**
```python
def extract_circuit(trajectory):
    """Convert WaveGen trajectory to LoRA circuit"""

    # Identify which neurons activated strongly
    active_neurons = []
    for step in trajectory:
        for layer_name, activations in step.activations.items():
            # Threshold: activation > mean + 2*std
            threshold = activations.mean() + 2 * activations.std()
            active = (activations > threshold).nonzero()
            active_neurons.append((layer_name, active))

    # Build circuit graph
    circuit = Circuit()
    circuit.nodes = unique(active_neurons)

    # Edges = strong connections (high weight magnitude)
    for (layer_i, neuron_i) in circuit.nodes:
        for (layer_j, neuron_j) in circuit.nodes:
            if layer_j > layer_i:  # only forward connections
                weight = get_weight(layer_i, neuron_i, layer_j, neuron_j)
                if abs(weight) > weight_threshold:
                    circuit.edges.append((neuron_i, neuron_j))

    return circuit
```

---

### 4. **Cache Invalidation When LoRA Updates**

When slow LoRA layers change, cached trajectories become stale.

**Problem:**
```
LoRA_domain updates (new knowledge added)
    ↓
Old cache entries no longer valid (based on old LoRA state)
    ↓
Must invalidate cache
```

**Solution: Provenance-based invalidation**

```python
class WaveGenCache:
    def invalidate_on_lora_update(self, updated_layers):
        """Called when LoRA layers change"""

        for entry in self.cache.entries:
            # Check if entry depends on updated layers
            if entry.depends_on(updated_layers):
                # Invalidate
                self.cache.evict(entry)
```

**Optimization: Tiered invalidation**

```
If LoRA_foundation updates:
    → Invalidate ALL cache (foundation affects everything)

If LoRA_domain updates:
    → Invalidate domain-specific cache only

If LoRA_swarm updates:
    → Invalidate swarm-specific cache

If LoRA_agent updates:
    → Invalidate only that agent's cache

If LoRA_context updates:
    → No invalidation (context changes every request anyway)
```

---

### 5. **Multi-Tier WaveGen Cache**

Instead of one monolithic cache, WaveGen can have **tiered caches** aligned with LoRA layers.

**Architecture:**

```python
class MultiTierWaveGenCache:
    def __init__(self):
        self.foundation_cache = TrajectoryCache(
            max_size=1000,
            eviction_policy='LRU',
            lora_dependency=['foundation']
        )

        self.domain_cache = TrajectoryCache(
            max_size=5000,
            eviction_policy='LRU',
            lora_dependency=['foundation', 'domain']
        )

        self.swarm_cache = TrajectoryCache(
            max_size=10000,
            eviction_policy='weighted_LRU',
            lora_dependency=['foundation', 'domain', 'swarm']
        )

        self.agent_cache = TrajectoryCache(
            max_size=20000,
            eviction_policy='weighted_LRU',
            lora_dependency=['foundation', 'domain', 'swarm', 'agent']
        )

        # Context layer: no cache (too fast-changing)

    def lookup(self, prompt, lora_snapshot):
        """Try caches from deepest to shallowest"""

        # Try foundation cache first (most stable, universal patterns)
        if hit := self.foundation_cache.lookup(prompt, lora_snapshot):
            return hit

        # Try domain cache (domain-specific patterns)
        if hit := self.domain_cache.lookup(prompt, lora_snapshot):
            return hit

        # Try swarm cache (team patterns)
        if hit := self.swarm_cache.lookup(prompt, lora_snapshot):
            return hit

        # Try agent cache (personal patterns)
        if hit := self.agent_cache.lookup(prompt, lora_snapshot):
            return hit

        # No hit
        return None

    def store(self, trajectory, lora_snapshot):
        """Store in appropriate tier based on universality"""

        # Compute universality score
        universality = compute_universality(trajectory, lora_snapshot)

        if universality > FOUNDATION_THRESHOLD:
            self.foundation_cache.store(trajectory)
        elif universality > DOMAIN_THRESHOLD:
            self.domain_cache.store(trajectory)
        elif universality > SWARM_THRESHOLD:
            self.swarm_cache.store(trajectory)
        else:
            self.agent_cache.store(trajectory)
```

**Benefit:**
- Stable patterns cached at deep tiers (rarely invalidated)
- Volatile patterns cached at shallow tiers (frequently updated)
- Cache hit rate improves (better alignment with LoRA dynamics)

---

## Updated WaveGen Components

### `WaveController` (with LoRA awareness)

```python
class WaveController:
    def __init__(self, backend, lora_hierarchy):
        self.backend = backend
        self.lora_hierarchy = lora_hierarchy  # NEW
        self.cache = MultiTierWaveGenCache()  # NEW: tiered
        self.circuit_detector = CircuitDetector()  # NEW

    def generate(self, prompt, max_tokens):
        # 1. Create LoRA snapshot
        lora_snapshot = self.lora_hierarchy.snapshot()

        # 2. Try cache lookup (with LoRA provenance)
        cache_key = hash(prompt + lora_snapshot.hash())

        if cached := self.cache.lookup(cache_key):
            # JUMP: use cached block
            self.cache.record_hit(cache_key)
            return cached.text

        # 3. Generate (no cache hit)
        trajectory = self._generate_with_lora(prompt, lora_snapshot, max_tokens)

        # 4. Cache if coherent
        if coherence(trajectory) > threshold:
            self.cache.store(trajectory, lora_snapshot)

        # 5. Extract circuit for LoRA
        if quality(trajectory) > threshold:
            circuit = self.circuit_detector.extract(trajectory)
            self.lora_hierarchy.context.consolidate(circuit)

        return trajectory.text

    def _generate_with_lora(self, prompt, lora_snapshot, max_tokens):
        """Generation loop with LoRA composition"""

        trajectory = []
        context = self.backend.tokenize(prompt)

        for step in range(max_tokens):
            # Forward through LoRA stack
            logits = self.backend.forward_with_lora(context, lora_snapshot)

            token = sample(logits)
            trajectory.append(StepObservation(
                token_id=token,
                logits=logits,
                activations=self.backend.get_activations(),  # NEW
                step_index=step
            ))

            context = append(context, token)

        return Trajectory(steps=trajectory)

    def consolidation_tick(self):
        """Periodic: check cache hit rates, signal LoRA"""

        for tier_name, tier_cache in self.cache.tiers.items():
            for entry in tier_cache.high_hit_rate():
                # Extract circuit
                circuit = self.circuit_detector.extract(entry.trajectory)

                # Determine target LoRA layer
                if tier_name == 'foundation':
                    target = 'foundation'
                elif tier_name == 'domain':
                    target = 'domain'
                elif tier_name == 'swarm':
                    target = 'swarm'
                else:
                    target = 'agent'

                # Request migration
                self.lora_hierarchy.request_consolidation(
                    circuit=circuit,
                    target_layer=target
                )

                # Evict from cache
                tier_cache.evict(entry)
```

---

### `BaseLLMBackend` (LoRA composition support)

```python
class BaseLLMBackend:
    def forward_with_lora(self, input_ids, lora_snapshot):
        """Forward pass through base + LoRA stack"""

        # Embed
        hidden = self.model.embed(input_ids)

        # Apply LoRA layers in order
        for layer_name in ['foundation', 'domain', 'swarm', 'agent', 'context']:
            if layer_name in lora_snapshot:
                lora_layer = lora_snapshot[layer_name]
                hidden = lora_layer.forward(hidden)

        # LM head
        logits = self.model.lm_head(hidden)

        return logits

    def get_activations(self):
        """Return intermediate activations (for circuit extraction)"""
        return self.model.activation_cache  # hooks set during forward pass
```

---

## Updated Data Flow

### Lifecycle with LoRA Integration

```
1. User Request
    ↓
2. Create LoRA Snapshot (freeze current state)
    ↓
3. WaveGen Cache Lookup (with LoRA provenance)
    ├─ HIT: Return cached block, record hit
    └─ MISS: Continue to generation
    ↓
4. Generate with LoRA Stack
    Base → Foundation → Domain → Swarm → Agent → Context → Output
    Record activations for circuit extraction
    ↓
5. WaveGen Cache Store (if coherent)
    Determine tier (foundation/domain/swarm/agent)
    ↓
6. Circuit Extraction
    Identify active neurons, strong connections
    ↓
7. LoRA Context Update
    Consolidate circuit in Context layer
    ↓
8. Consolidation Tick (periodic)
    Check cache hit rates
    High hit rate → Signal migration to LoRA
    Evict from cache once consolidated
    ↓
9. LoRA Migration (Context → Agent → Swarm → Domain)
    Autocorrelation detector triggers upward migration
    Bandwidth limits prevent avalanche
    ↓
10. Cache Invalidation (if slow layers updated)
    Foundation/Domain update → invalidate dependent caches
```

---

## Configuration

### WaveGen Config (with LoRA)

```python
wavegen_config = {
    'cache': {
        'foundation_tier': {
            'max_size': 1000,
            'lora_dependencies': ['foundation'],
            'eviction_policy': 'LRU',
            'consolidation_threshold': 0.9  # 90% hit rate → migrate to LoRA
        },
        'domain_tier': {
            'max_size': 5000,
            'lora_dependencies': ['foundation', 'domain'],
            'eviction_policy': 'LRU',
            'consolidation_threshold': 0.85
        },
        'swarm_tier': {
            'max_size': 10000,
            'lora_dependencies': ['foundation', 'domain', 'swarm'],
            'eviction_policy': 'weighted_LRU',
            'consolidation_threshold': 0.7
        },
        'agent_tier': {
            'max_size': 20000,
            'lora_dependencies': ['foundation', 'domain', 'swarm', 'agent'],
            'eviction_policy': 'weighted_LRU',
            'consolidation_threshold': 0.5
        }
    },
    'circuit_extraction': {
        'activation_threshold': 'mean + 2*std',
        'weight_threshold': 0.1,
        'min_circuit_size': 5  # neurons
    },
    'consolidation_tick_interval': 100  # steps
}
```

---

## Benefits of Integration

### 1. **Two-Tier Memory**
- **WaveGen:** Fast retrieval (cache hit = instant)
- **LoRA:** Adaptive parameters (model learns over time)

### 2. **Cache Efficiency**
- Patterns internalized in LoRA don't need caching
- Cache space freed for new patterns
- Higher hit rate (tiered cache aligned with LoRA)

### 3. **Interpretability**
- WaveGen trajectories → circuits
- Circuits traceable (which neurons, which function)
- Migrations logged (when pattern moved to which layer)

### 4. **No Manual Tuning**
- Cache hit rate automatically signals consolidation
- Autocorrelation detects stable patterns
- Bandwidth limits prevent overload

### 5. **Scalability**
- Shared LoRA layers (Swarm, Domain) benefit all agents
- Each agent has personal cache + personal Agent layer
- Knowledge accumulates without duplication

---

## Example Scenario

**Task:** Write Python JSON parser (repeated across multiple agents)

```
Agent A, Request 1:
    ├─ WaveGen: Cache miss
    ├─ Generate: "import json\ndef parse_json..."
    ├─ Cache: Store in agent tier
    └─ LoRA: Context layer learns weak circuit

Agent A, Request 2 (same task):
    ├─ WaveGen: Cache hit! (instant return)
    ├─ Cache hit rate: 1/2 = 50%
    └─ Circuit: Context → SCREENED (phase transition)

Agent A, Requests 3-5:
    ├─ WaveGen: Cache hit
    ├─ Cache hit rate: 4/5 = 80%
    └─ Consolidation: Hit rate > threshold → migrate to Agent LoRA

Agent B, Request 1 (same task):
    ├─ WaveGen: Cache miss (different agent)
    ├─ Generate: Similar pattern
    ├─ LoRA Agent update: Resonance with Agent A
    └─ Swarm layer: Begins to consolidate pattern

Agents A+B, Requests 10+:
    ├─ WaveGen: Swarm tier cache hit (shared)
    ├─ Hit rate: 95%
    ├─ Consolidation: Migrate to LoRA_swarm
    └─ Cache: Evict (no longer needed, pattern in weights)

Future agents C, D, E:
    ├─ Inherit: LoRA_swarm contains JSON parsing circuit
    ├─ First request: Already "knows" the pattern
    └─ No cache needed (internalized in model)
```

**Result:**
- First agent: Learning phase (cache + LoRA context)
- Multiple agents: Consolidation (cache → LoRA swarm)
- Future agents: Immediate knowledge (inherited from swarm)

---

## Summary

**WaveGen** = Working memory (fast, volatile, trajectory-based)
**DynamicLoRA** = Long-term memory (slower, stable, circuit-based)

**Integration** creates a complete adaptive system:
- Cache optimizes inference speed
- LoRA enables continuous learning
- Together: fast + adaptive + interpretable

**Key mechanisms:**
1. Provenance hash (cache ↔ LoRA coordination)
2. Circuit extraction (trajectories → activation patterns)
3. Hit rate signaling (cache → LoRA consolidation trigger)
4. Tiered architecture (aligned with LoRA hierarchy)
5. Cache invalidation (when LoRA updates)

**This is how biological memory works:** Working memory (hippocampus) → Long-term memory (cortex), with pattern replay during consolidation.

WaveGen + DynamicLoRA = artificial hippocampus + cortex.
