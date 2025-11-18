# Intelexta Schemas & Protocol
### An Open Standard for Verifiable AI Workflows
A project by [Gauge Freedom, Inc.](https://gaugefreedom.com)

This directory contains the official JSON Schema definitions for **IntelexTA Content-Addressable Receipts (CARs)**.

A **CAR** is a portable, tamper-evident receipt for an AI-assisted workflow run. It captures:
- **What** ran (model, prompts, steps),
- **How** it ran (checkpoints, hashes, signatures),
- **What it cost** (tokens, USD, nature cost),
- And **how well it can be replayed** (proof modes like `process`, `exact`, `semantic`).

These schemas are the *source of truth* for any tool that wants to **emit, verify, or analyze** IntelexTA CARs.

---

## Current Schema Version

**`car-v0.3.schema.json`** – Active schema (November 2025)

This version accurately reflects the current implementation of:
- IntelexTA Desktop CAR exporter (CAR-Full)
- Verifiable Summary MCP server (CAR-Lite)
- `intelexta-verify` CLI verifier
- Web verifier

If you are building anything new, **target v0.3**.

---

## Schema Versions

### v0.3 (Current)

**Status**: ✅ Active  
**File**: `car-v0.3.schema.json`  
**Release**: November 2025

**Key Features**:
- Includes `proof.process.sequential_checkpoints` structure
- Supports three `match_kind` modes: `"process"`, `"exact"`, `"semantic"`
- Uses hybrid naming convention (top-level: snake_case, `run.steps`: camelCase)
- Validates both Desktop CARs and MCP CAR-Lite bundles
- 100% backward compatible with existing implementations

**What's New**:
- Added `proof.process` and `process_checkpoint_proof` definitions
- Updated `run_step` field names to camelCase (matches actual CARs)
- Added optional `policy_ref.model_catalog_hash` and `model_catalog_version`
- Relaxed `checkpoints` pattern to accept both `ckpt:`-prefixed IDs and plain UUIDs
- Added conditional validation for `match_kind: "process"`

### v0.2 (Historical)

**Status**: 📚 Reference only  
**File**: `car-v0.2.schema.json`  
**Release**: Initial design

**Limitations**:
- Missing `proof.process` structure (implementation diverged)
- Used snake_case for `run_step` fields (actual CARs use camelCase)
- Strict `ckpt:` prefix requirement for checkpoint IDs

New code **should not** target v0.2; it is kept only so old discussions and drafts remain interpretable.

---

## Validation

### Quick Validation (No Extra Dependencies)

```bash
node validate-car.js examples/sample.car.json
````

This performs basic structural checks using the v0.3 schema.

### Full Schema Validation (Recommended)

Install dependencies:

```bash
npm install ajv ajv-formats
```

Validate against schema:

```bash
node -e "
const Ajv = require('ajv/dist/2020');
const addFormats = require('ajv-formats');
const fs = require('fs');

const ajv = new Ajv({ allErrors: true, strict: false });
addFormats(ajv);

const schema = JSON.parse(fs.readFileSync('car-v0.3.schema.json', 'utf8'));
const car = JSON.parse(fs.readFileSync('your-car.json', 'utf8'));

const validate = ajv.compile(schema);
if (validate(car)) {
  console.log('✓ Valid CAR');
} else {
  console.error('✗ Invalid:', validate.errors);
}
"
```

### Validate ZIP Bundles

Extract and validate:

```bash
unzip -p bundle.car.zip car.json > temp.car.json
node validate-car.js temp.car.json
```

Or use `intelexta-verify` CLI:

```bash
intelexta-verify bundle.car.zip
```

---

## CAR Structure Overview

High-level shape of a v0.3 CAR:

```json
{
  "id": "car:<sha256>",
  "run_id": "...",
  "created_at": "2025-11-17T...",
  "run": {
    "kind": "exact" | "concordant" | "interactive",
    "steps": [
      {
        "id": "...",
        "runId": "...",           // camelCase
        "orderIndex": 0,          // camelCase
        "checkpointType": "...",  // camelCase
        "tokenBudget": 1000,      // camelCase
        "proofMode": "exact",     // camelCase
        ...
      }
    ]
  },
  "proof": {
    "match_kind": "process" | "exact" | "semantic",
    "process": {                 // Present when match_kind === "process"
      "sequential_checkpoints": [
        {
          "id": "...",
          "prev_chain": "",
          "curr_chain": "<sha256>",
          "signature": "<base64>",
          "run_id": "...",        // snake_case
          "kind": "Step",
          "timestamp": "...",
          "inputs_sha256": "...", // snake_case
          "outputs_sha256": "...",
          "usage_tokens": 0,
          "prompt_tokens": 0,
          "completion_tokens": 0
        }
      ]
    }
  },
  "policy_ref": { ... },
  "budgets": { ... },
  "provenance": [ ... ],
  "checkpoints": [ ... ],
  "sgrade": { ... },
  "signer_public_key": "...",
  "signatures": [ ... ]
}
```

---

## Naming Convention (Hybrid)

v0.3 uses a **hybrid naming convention** that matches the actual implementation:

| Location                               | Convention | Example                            |
| -------------------------------------- | ---------- | ---------------------------------- |
| Top-level fields                       | snake_case | `run_id`, `created_at`             |
| `run.steps` fields                     | camelCase  | `runId`, `orderIndex`, `proofMode` |
| `proof.process.sequential_checkpoints` | snake_case | `prev_chain`, `inputs_sha256`      |

This is **intentional** and consistent across both Desktop and MCP implementations.

---

## `match_kind` Semantics

These modes describe how a CAR is intended to be *replayed or checked*:

### `"process"`

* **Used by**: CAR-Lite (MCP), Desktop interactive workflows
* **Meaning**: Verify the **cryptographic process**:

  * Hash chain over checkpoints
  * Signatures over chain and body
* **Requires**: `proof.process.sequential_checkpoints`
* **Does NOT require**: Replaying the model outputs.

### `"semantic"`

* **Used by**: Desktop concordant/semantic workflows
* **Meaning**: Verify via **semantic similarity** metrics (rather than exact bytes).
* **Current status**:

  * Desktop sets `match_kind: "semantic"`
  * Future versions will enforce presence of:

    * `epsilon`
    * `distance_metric`
    * `original_semantic_digest`

### `"exact"`

* **Used by**: Desktop exact replay workflows
* **Meaning**: Target **byte-for-byte deterministic replay**.
* `proof.process` may still be present to capture the signed process trace.

---

## How Other Projects Should Use This

If you are:

* A **journal** (e.g., Gauge Freedom Journal),
* A **tooling vendor**,
* Or a **researcher** building reproducible pipelines,

then:

1. Treat `car-v0.3.schema.json` as the canonical contract for CAR artifacts.
2. Accept CARs from authors or tools that validate against this schema.
3. Use `intelexta-verify` or your own verifier to:

   * Validate structure (schema)
   * Validate signatures and hash chains
   * Optionally, implement your own replay/semantics on top.

You do **not** have to use Intelexta Desktop to participate; any system emitting CARs that conform to v0.3 is considered a first-class citizen of the IntelexTA trust layer.

---

## Development Workflow

### Creating a New Schema Version

1. **Discovery Phase**

   * Analyze actual CAR outputs from all implementations.
   * Document field names, types, and structures.
   * Identify gaps between schema and reality.

2. **Schema Design**

   * Create `car-vX.Y.schema.json` based on findings.
   * Add new definitions to `$defs`.
   * Update conditional validation rules.

3. **Validation**

   * Test against sample CARs from Desktop and MCP.
   * Ensure backward compatibility where possible.
   * Document any breaking changes.

4. **Documentation**

   * Update this README.
   * Add a migration guide if needed.
   * Update downstream tools and references.

### Testing Schema Changes

```bash
# Validate Desktop CAR
node validate-car.js path/to/desktop.car.json

# Validate MCP CAR
unzip -p mcp-bundle.car.zip car.json > temp.car.json
node validate-car.js temp.car.json

# Run CLI verifier (crypto + structure)
intelexta-verify desktop-bundle.car.zip
intelexta-verify mcp-bundle.car.zip
```

---

## Documentation

* **`PHASE1_DISCOVERY_FINDINGS.md`**
  Detailed analysis of the current CAR implementation (Desktop + MCP + verifier).

* **`PHASE2_SCHEMA_IMPLEMENTATION.md`**
  v0.3 schema creation process and validation results.

For additional context, see:

* `apps/verifiable-summary/CAR_FORMAT.md` – CAR format guide for the MCP server
* `apps/verifiable-summary/CAR_LITE_PLAN.md` – CAR-Lite profile specification

---

## References

### Implementation Files

* Desktop exporter: `src-tauri/src/car.rs`
* MCP server: `apps/verifiable-summary/server/src/provenance.ts`
* CLI verifier: `src-tauri/crates/intelexta-verify/src/main.rs`

These should always remain consistent with `car-v0.3.schema.json`. If they diverge, the schema and implementations must be reconciled.

---

## Contributing

When proposing schema changes:

1. Ensure the change reflects **actual implementation behavior** (not just an aspirational design).
2. Test against **real CARs** from both Desktop and MCP.
3. Document the rationale in a Phase document (e.g., `PHASE3_*.md`).
4. Update `validate-car.js` and any automated tests.
5. Consider backward compatibility and migration for existing CARs.

---

## License

See the main IntelexTA repository `LICENSE` file.

---
Learn more about our mission and products at **[gaugefreedom.com](https://gaugefreedom.com)**.
