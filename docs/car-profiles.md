# CAR Profiles: Lite vs Full

This document explains the two **profiles** built on top of the Intelexta CAR v0.3 schema:

- **CAR-Lite** – Minimal, easy-to-emit profile for plugins and simple tools  
- **CAR-Full** – Rich evidence profile used by Intelexta Desktop and production workflows

Both profiles:

- Conform **exactly** to `schemas/car-v0.3.schema.json`
- Are verifiable with the same tools (`intelexta-verify` CLI, web-verifier)
- Produce portable, tamper-evident receipts for AI-assisted runs

The difference is **how much evidence** they carry and **how hard they are to integrate**.

---

## 1. Common Ground: CAR v0.3

Regardless of profile, every valid CAR:

- Is a JSON object matching `car-v0.3.schema.json`
- Has the same top-level structure:

  ```json
  {
    "id": "car:<sha256>",
    "run_id": "...",
    "created_at": "...",
    "run": { ... },
    "proof": { ... },
    "policy_ref": { ... },
    "budgets": { ... },
    "provenance": [ ... ],
    "checkpoints": [ ... ],
    "sgrade": { ... },
    "signer_public_key": "...",
    "signatures": [ ... ]
  }
````

* Uses the same hybrid naming convention:

  * Top-level and checkpoints: **snake_case** (e.g. `run_id`, `created_at`, `inputs_sha256`)
  * `run.steps[*]`: **camelCase** (e.g. `runId`, `orderIndex`, `checkpointType`, `tokenBudget`, `proofMode`)

The v0.3 schema is the **only** contract you need to validate both CAR-Lite and CAR-Full.

---

## 2. CAR-Lite Profile

**Goal:** Make it trivial for external tools (MCP servers, LangChain pipelines, notebooks, etc.) to produce *valid, verifiable* CARs with **minimal instrumentation**.

CAR-Lite is what the **Verifiable Summary MCP** emits.

### 2.1. Typical Use Case

* Single-step or very simple workflows
* Summarization, classification, or basic analysis
* Limited access to internal runtime metrics (no detailed cost model, no semantic replay)

Example: “Generate a verifiable summary of this text” → one step, one checkpoint, one CAR.

### 2.2. Characteristic Choices

CAR-Lite chooses *safe defaults* for unknown or hard-to-measure quantities.

* `run.kind`: usually `"concordant"`

* `proof.match_kind`: **always `"process"`**

  * The CAR proves the *process chain* (hashes + signatures), not replay semantics.

* `proof.process.sequential_checkpoints`:

  * At least **one** checkpoint entry
  * Fields populated from whatever metrics the tool has (often just hashes & timestamps)

* `budgets`:

  * May be **0** if costs/tokens are unknown:

    ```json
    "budgets": {
      "usd": 0,
      "tokens": 0,
      "nature_cost": 0
    }
    ```

* `provenance`:

  * Minimum 3 claims is typical:

    * `config` (policy/config/hash)
    * `input` (hash of source content)
    * `output` (hash of model output)

* `checkpoints`:

  * Often a single ID, commonly with `ckpt:` prefix:

    ```json
    "checkpoints": ["ckpt:<run_id>"]
    ```

* `policy_ref`:

  * Uses a static or simple policy:

    ```json
    "policy_ref": {
      "hash": "sha256:<policy-doc-hash>",
      "egress": true,
      "estimator": "usage_tokens * 0.010000 nature_cost/token"
    }
    ```

* `sgrade`:

  * Uses a baseline score such as 70–90 with neutral component weights:

    ```json
    "sgrade": {
      "score": 85,
      "components": {
        "provenance": 1.0,
        "energy": 1.0,
        "replay": 0.8,
        "consent": 0.8,
        "incidents": 1.0
      }
    }
    ```

* `signer_public_key` / `signatures`:

  * **Production**: should use real Ed25519 keys (`ed25519-body:…`, `ed25519-checkpoint:…`)
  * **Development**: can use dummy or “unsigned” signatures, as long as they match the string pattern; verifiers will then mark the CAR as unsigned / untrusted.

### 2.3. When to Use CAR-Lite

Choose CAR-Lite if:

* You’re building a **plugin or integration** and want to ship quickly.
* You don’t have token-level or cost-level metrics yet.
* You want to attach a cryptographic “receipt” to outputs, but full replay semantics are out of scope (for now).

CAR-Lite is perfect for:

* Journals that just need *proof a summary was generated as claimed*
* Simple MCP servers
* Scripts and notebooks that want verifiable receipts with minimal code

---

## 3. CAR-Full Profile

**Goal:** Provide **rich, audit-level evidence** for complex workflows, including detailed budgets, multiple steps, and replay semantics.

CAR-Full is what **Intelexta Desktop** emits for LLM workflows.

### 3.1. Typical Use Case

* Multi-step workflows (prompt → tool → summarizer → post-processor)
* Research pipelines, production systems, or regulated environments
* Need to reason about:

  * Token usage / USD cost
  * Nature cost (gCO₂e-equivalent)
  * Replay mode (`exact` / `semantic`)
  * Multiple checkpoints over time

### 3.2. Characteristic Choices

CAR-Full uses the same schema but commits to filling in more of it.

* `run.kind`:

  * `"exact"` for deterministic/replayable runs
  * `"concordant"` for semantic/approximate replay

* `proof.match_kind`:

  * `"exact"` or `"semantic"` (and may still include a `process` section)
  * `process` is still present to preserve cryptographic trace

* `run.steps`:

  * Multiple steps with detailed config:

    * `stepType` (e.g. `"llm"`, `"prompt"`, `"summarize"`)
    * `tokenBudget`
    * `epsilon` for concordant proof
    * `configJson` capturing richer configuration

* `proof.process.sequential_checkpoints`:

  * One checkpoint **per step** (or more, for interactive runs)
  * Each checkpoint includes:

    * `inputs_sha256`, `outputs_sha256`
    * `usage_tokens`, `prompt_tokens`, `completion_tokens`
    * `timestamp`, `kind`, `run_id`
  * Hash chain and signatures match what the CLI verifier checks.

* `budgets`:

  * Populated from real accounting:

    * `usd` from provider pricing
    * `tokens` from model usage
    * `nature_cost` from a concrete estimator

* `policy_ref`:

  * Often includes Desktop-specific model governance fields:

    ```json
    "policy_ref": {
      "hash": "sha256:<policy-doc>",
      "egress": true,
      "estimator": "usage_tokens * 0.010000 nature_cost/token",
      "model_catalog_hash": "sha256:<catalog>",
      "model_catalog_version": "1.0.0"
    }
    ```

* `provenance`:

  * Richer set of claims:

    * Multiple inputs/outputs
    * References to datasets, model cards, configuration artifacts
    * Possibly incident logs or other audit anchors

### 3.3. When to Use CAR-Full

Choose CAR-Full if:

* You control the full stack (e.g., a desktop or server application).
* You need **strong replay guarantees** or detailed cost accounting.
* You’re aiming at **auditable, long-lived workflows** (research, finance, legal, etc.).
* You want to expose more detailed stewardship metrics (S-Grade).

CAR-Full is ideal for:

* Intelexta Desktop projects
* Laboratories and institutions standardizing on verifiable AI
* Systems that might be audited years later for reproducibility

---

## 4. Quick Comparison

| Aspect                 | CAR-Lite                      | CAR-Full                                 |
| ---------------------- | ----------------------------- | ---------------------------------------- |
| Schema                 | `car-v0.3`                    | `car-v0.3`                               |
| Typical producer       | MCP / plugin / simple tool    | Intelexta Desktop, rich apps             |
| Steps                  | Usually 1                     | 1+ (multi-step workflows)                |
| `match_kind`           | `"process"`                   | `"exact"` or `"semantic"` (plus process) |
| Budgets                | 0 or coarse approximations    | Real token/USD/nature metrics            |
| Provenance             | Minimal (config/input/output) | Rich (datasets, configs, more outputs)   |
| Checkpoints            | 1                             | Multiple per run                         |
| Replay focus           | Cryptographic trace only      | Trace + semantic/exact replay            |
| Integration complexity | Low                           | Medium–High                              |
| Best for               | Plugins, quick integrations   | Desktop, research, regulated workflows   |

Both profiles are **first-class citizens**: they differ in *density*, not in legitimacy.

---

## 5. Migrating from CAR-Lite to CAR-Full

Many integrators will start with CAR-Lite and later add more detail.

A natural migration path:

1. **Phase 1 – CAR-Lite**

   * Emit one-step CARs with `match_kind: "process"`.
   * Fill `provenance` with config/input/output hashes.
   * Keep budgets at 0 if you don’t have metrics yet.

2. **Phase 2 – CAR-Lite+**

   * Start populating `budgets.tokens` and `budgets.usd`.
   * Add more `provenance` claims (datasets, extra artifacts).
   * Add more checkpoints if your workflow has multiple steps.

3. **Phase 3 – CAR-Full**

   * Implement multi-step `run.steps` mapped to your pipeline.
   * For exact/semantic replay:

     * Set `run.kind` and `proof.match_kind` to `"exact"` or `"semantic"`.
     * Populate semantic digests once your replay engine is ready.
   * Align with the same patterns Intelexta Desktop uses.

At every stage, your CARs remain valid v0.3, and verifiers don’t break—only the *strength* and *richness* of the proof improves.

---

## 6. Validation & Tooling

Both profiles are validated the same way:

* **Schema check** against `schemas/car-v0.3.schema.json`
* **Cryptographic check** via:

  * `intelexta-verify bundle.car.zip`
  * The web verifier (drop the `.car.zip`)

Consumers (journals, auditors, other tools) don’t need to distinguish explicitly between “Lite” and “Full”: they can simply **inspect what’s actually present** in the CAR and decide how much weight to give it.

The profile concept is primarily for **emitters**: it tells you how much you’re committing to implement.

