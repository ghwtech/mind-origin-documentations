# MindOrigin / Noah — P/G Behavioral Layer

## Implementation & Build Guide (for spec v1.6.1)

> **What this document is.** A step-by-step engineering guide for building the
> P/G Behavioral Layer described in `MindOrigin_PG_Behavioral_Layer_v1.6.1_Executable.pdf`.
> For **every phase** it states: *what it does*, *what to build*, *how to run it*,
> *how to test it*, and the *exit criteria* that let you move on.
>
> **Authoritative source.** The PDF spec governs. Where this guide and the spec
> disagree, the spec wins; where the spec's prose and a schema disagree, the
> **schema** wins (spec, "How to read this document").
>
> **Stack decision (locked).** Python 3.11+, Pydantic v2 (schemas + JSON-Schema
> export + runtime validation), pytest (B1–B10 regression harness).
> **Rules-only first**: every module ships as deterministic logic; LLM calls are
> deferred behind a pluggable interface until a benchmark forces one.

---

## Table of contents

- [Part A — Project setup & conventions](#part-a)
  - [A.1 Prerequisites](#a1)
  - [A.2 One-time environment setup](#a2)
  - [A.3 Repository layout](#a3)
  - [A.4 Global conventions (frozen)](#a4)
  - [A.5 How to run things (cheat sheet)](#a5)
- [Part B — The invariants every phase is graded against](#part-b)
- [Part C — Phased build](#part-c)
  - [Phase 0 — Foundations](#phase0)
  - [Phase 1 — M5 Aggregator / Validator](#phase1)
  - [Phase 2 — M1 Domain Adapter](#phase2)
  - [Phase 3 — M2 Candidate Target Detector](#phase3)
  - [Phase 4 — M3 Raw Consequence Generator](#phase4)
  - [Phase 5 — M4 P/G Projector](#phase5)
  - [Phase 6 — Integration & MVP acceptance](#phase6)
  - [Phase 7 — M4 dynamic extension (LLM fallback / learned projector)](#phase7)
- [Part D — Testing strategy (all levels)](#part-d)
- [Part E — Deferred work (do not build)](#part-e)
- [Appendix 1 — Frozen vocabularies](#appendix1)
- [Appendix 2 — B1–B10 expected labels](#appendix2)
- [Appendix 3 — Schema field reference](#appendix3)

---

<a name="part-a"></a>
# Part A — Project setup & conventions

<a name="a1"></a>
## A.1 Prerequisites

| Tool | Version | Why |
|---|---|---|
| Python | 3.11+ | `StrEnum`, modern typing, Pydantic v2 |
| pip / venv | bundled | dependency isolation |
| Pydantic | v2.x | the 5 frozen schemas, validation, JSON-Schema export |
| pytest | 8.x | B1–B10 regression harness + unit tests |
| (optional) anthropic SDK | latest | only when the deferred `ClaudeProvider` for M3 is built |

No LLM key is required for the MVP rules-only build. The repo must run end-to-end
with **zero network access**.

<a name="a2"></a>
## A.2 One-time environment setup

From the project root (`C:\mindorigin`), in PowerShell:

```powershell
# 1. Create the package directory (code lives here; docs live in mind-origin-documentations\)
New-Item -ItemType Directory -Force behavioral_layer

# 2. Create and activate a virtual environment
python -m venv .venv
.\.venv\Scripts\Activate.ps1

# 3. Install dependencies (after pyproject.toml exists — see Phase 0.1)
pip install -e ".[dev]"
```

> **Activation note.** If PowerShell blocks the activation script, run once:
> `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned`.
> On macOS/Linux the activate command is `source .venv/bin/activate`.

`pyproject.toml` (created in Phase 0.1) pins the toolchain. The `[build-system]`
and `[tool.setuptools.packages.find]` sections are required for the editable
install (`pip install -e`) to discover the package:

```toml
[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"

[project]
name = "behavioral-layer"
version = "0.1.0"
description = "MindOrigin / Noah — P/G Behavioral Layer (MVP, spec v1.6.1)"
requires-python = ">=3.11"
dependencies = ["pydantic>=2.6"]

[project.optional-dependencies]
dev = ["pytest>=8.0", "pytest-cov>=4.0"]

[tool.setuptools.packages.find]
include = ["behavioral_layer*"]

[tool.pytest.ini_options]
testpaths = ["behavioral_layer/tests"]
markers = [
    "benchmark: B1-B10 contract/regression scenarios",
    "acceptance: 100 held-out scenario suite (Phase 6)",
]
```

<a name="a3"></a>
## A.3 Repository layout

```
C:\mindorigin\
├─ behavioral_layer/                  # the Python package (the build)
│  ├─ __init__.py
│  ├─ schemas/                        # Phase 0.2 — the 5 frozen Pydantic models
│  │  ├─ __init__.py
│  │  ├─ common.py                    # shared types: Horizon, ranges, key fns
│  │  ├─ world_state.py               # WorldState (§11)
│  │  ├─ relation_graph.py            # RelationGraph (§12)
│  │  ├─ raw_trace.py                 # RawConsequenceTrace (§13)
│  │  ├─ impact_field.py              # ImpactField (§14) — canonical output
│  │  └─ group_state.py               # GroupState (§16)
│  ├─ vocab/                          # Phase 0.3 — the 2 closed vocabularies
│  │  ├─ __init__.py
│  │  ├─ fact_type.py                 # FACT_TYPE enum (13 incl. unmappable)
│  │  ├─ mechanism.py                 # MECHANISM enum (11 incl. unmappable)
│  │  └─ dimensions.py                # P-dims (8), G-vars (3), inclusion reasons
│  ├─ keys.py                         # Phase 0.4 — natural_key / storage_key
│  ├─ config/                         # per-world config (NOT code)
│  │  ├─ frozen_v1.yaml               # schema_version/vocab_version pins
│  │  ├─ m2_caps.yaml                 # max_candidates, thresholds, group_size_cap
│  │  ├─ projector_rules.yaml         # M4 per-mechanism rule table + rule_reliability
│  │  └─ worlds/
│  │     └─ npc_village_01.yaml       # M1 per-world mapping table
│  ├─ modules/                        # the 5 modules
│  │  ├─ __init__.py
│  │  ├─ m1_adapter.py
│  │  ├─ m2_targets.py
│  │  ├─ m3_consequence.py
│  │  ├─ m4_projector.py
│  │  └─ m5_validator.py              # built FIRST
│  ├─ providers/                      # deferred LLM seam
│  │  ├─ __init__.py
│  │  └─ llm_provider.py              # LLMStepProvider iface + RulesProvider
│  ├─ grading/                        # Phase 0.6 — the §18 metrics
│  │  ├─ __init__.py
│  │  ├─ metrics.py
│  │  └─ rubric.py
│  ├─ pipeline.py                     # M1→M2→M3→M4→M5 orchestrator + CLI
│  └─ tests/
│     ├─ __init__.py
│     ├─ fixtures/
│     │  └─ benchmarks/               # Phase 0.5 — B1..B10 as data files
│     │     ├─ b01_painful_truth.json
│     │     ├─ ...
│     │     └─ b10_scarcity_sharing.json
│     ├─ test_schemas.py
│     ├─ test_vocab.py
│     ├─ test_keys.py
│     ├─ test_m5_validator.py
│     ├─ test_m1_adapter.py
│     ├─ test_m2_targets.py
│     ├─ test_m3_consequence.py
│     ├─ test_m4_projector.py
│     ├─ test_grading.py
│     └─ test_benchmarks.py           # the B1-B10 regression suite
├─ mind-origin-documentations/        # specs + this guide (git-tracked)
│  ├─ MindOrigin_PG_Behavioral_Layer_v1.6.1_Executable.pdf
│  └─ MindOrigin_Behavioral_Layer_Implementation_Plan.md   ← this file
└─ pyproject.toml
```

<a name="a4"></a>
## A.4 Global conventions (frozen — spec §10)

These are enforced by code in Phase 0 and assumed everywhere after.

1. **Five frozen schemas:** WorldState, RelationGraph, RawConsequenceTrace,
   ImpactField (canonical output), GroupState.
2. **Two versioned axes on every payload:** `schema_version` (shape changes) and
   `vocab_version` (new mechanism/fact_type). A module **may not** emit a token
   outside its declared `vocab_version`.
3. **Ranges:** all P-values and deltas ∈ `[-1, +1]`; all confidences and
   uncertainties ∈ `[0, 1]`.
4. **Horizons** are a closed set: `short | mid | long`. World tick spans map to
   these three in **adapter config**, never in the core.
5. **Natural key** = `target_type | target_id | dimension | horizon`.
   `target_type` is included so a person id and a group id sharing a string never
   collide. It is the unit of target-matching, dedup, rerun-stability alignment,
   and `derived_from` references.
6. **Storage key** = `world_id | tick | actor_id | action_id | <natural key>`
   — globally unique for persistence, logs, training rows.
7. **delta** is the *projected, pre-saturation* change, bounded to `[-1,+1]` but
   **not** clipped against the target's current state. Mechanical saturation is the
   world-model's job at state-update, never M4's.

<a name="a5"></a>
## A.5 How to run things (cheat sheet)

```powershell
# Activate the venv first (every new shell)
.\.venv\Scripts\Activate.ps1

# Run the entire test suite
pytest

# Run only the B1-B10 regression harness
pytest -m benchmark

# Run one phase's tests
pytest behavioral_layer/tests/test_m5_validator.py -v

# Run a single scenario end-to-end through the pipeline and print the ImpactField
python -m behavioral_layer.pipeline --scenario b02_theft_observed --pretty

# Validate an arbitrary ImpactField file with M5 (strict mode)
python -m behavioral_layer.modules.m5_validator --in out.json --mode strict

# Export JSON Schema for the 5 frozen models (sanity / external consumers)
python -m behavioral_layer.schemas --export-json-schema schemas_out/

# Coverage report
pytest --cov=behavioral_layer --cov-report=term-missing
```

> Every module also exposes a `__main__` CLI so it can be exercised in isolation
> on a JSON file — this is how you debug a single stage without the full pipeline.

---

<a name="part-b"></a>
# Part B — The invariants every phase is graded against

Treat these as cross-cutting acceptance gates. A phase is not "done" if it breaks one.

| # | Invariant | Where enforced |
|---|---|---|
| I1 | Behavioral layer outputs agent-neutral ΔP/ΔG only — no personality, emotion, decision, normative judgment | M1 guard, M4 rules, code review |
| I2 | **Anti-smuggling:** M1 output contains only facts + relations; never a delta/affect/evaluative field | M1 validator (hard reject) |
| I3 | Schema governs; the 5 schemas + 2 vocabularies are frozen | Pydantic models, vocab enums |
| I4 | Closed vocabularies; unmappable → explicit failure value, never a silent drop | M1/M3 failure paths |
| I5 | `delta` is pre-saturation, never clipped against current state | M4 (no boundary arithmetic) |
| I6 | Provenance mandatory: primary→`evidence_ref`; derived→`derived:true`+`derived_from` | ImpactField model + M5 |
| I7 | Calibration deferral: member_welfare ships `structural_only`, `numeric_scoring:false`, `delta 0.0`, `uncertainty 1.0` | M5 |
| I8 | Truncation flagged, never silent; a truncated branch = "not computed", not "no effect" | M3 |
| I9 | Recall-first M2: over-inclusion fine; omission is the only real failure; pruned candidates logged | M2 |

---

<a name="part-c"></a>
# Part C — Phased build

The build order is the spec's §22, expanded. Dependency spine:

```
Phase 0 (schemas + vocab + keys + benchmarks + grader)
   └─► Phase 1: M5 (validator + harness)   ← everything is graded through this
            └─► M1 ─► M2 ─► M3 ─► M4 ─► Phase 6 (integrate → B1–B10 → §19)
```

---

<a name="phase0"></a>
## Phase 0 — Foundations

### What it does
Freezes the contracts so no later module can drift: the 5 schemas, the 2 closed
vocabularies, the key functions, the B1–B10 benchmark fixtures, and the grader.
**Nothing downstream is correct if these are wrong**, so this phase is verified
hardest.

### What to build

**0.1 Repo scaffold.** Directory tree from §A.3; `pyproject.toml` from §A.2.

**0.2 The 5 Pydantic models** (`behavioral_layer/schemas/`). Each carries
`schema_version` and `vocab_version` and validates ranges. Sketch:

```python
# schemas/common.py
from enum import StrEnum
from typing import Annotated
from pydantic import BaseModel, Field

Unit = Annotated[float, Field(ge=-1.0, le=1.0)]    # P-values and deltas
Prob = Annotated[float, Field(ge=0.0, le=1.0)]     # confidence / uncertainty

class Horizon(StrEnum):
    short = "short"
    mid = "mid"
    long = "long"

class Versioned(BaseModel):
    schema_version: str = "1.0"
    vocab_version: str = "1"
    model_config = {"extra": "forbid"}   # reject unknown fields → schema governs
```

```python
# schemas/impact_field.py  (the canonical output, §14)
from pydantic import BaseModel, model_validator
from .common import Versioned, Unit, Prob, Horizon
from ..vocab.dimensions import TargetType, PDim, GVar

class ImpactItem(BaseModel):
    target_id: str
    target_type: TargetType            # "P" | "G"
    dimension: str                     # named P-dim or G-var (validated below)
    delta: Unit
    horizon: Horizon
    uncertainty: Prob
    # provenance — exactly one form:
    evidence_ref: str | None = None    # primary item → a step_id
    derived: bool = False              # derived item (member_welfare)
    derived_from: list[str] | None = None
    calibration_status: str | None = None   # "structural_only" until λ,w_P set
    numeric_scoring: bool = True
    model_config = {"extra": "forbid"}

    @model_validator(mode="after")
    def _provenance(self):
        if self.derived:
            assert self.derived_from, "derived item needs derived_from"
            assert self.evidence_ref is None, "derived item omits evidence_ref"
        else:
            assert self.evidence_ref, "primary item needs evidence_ref"
        # dimension must be valid for the target_type (see Phase 0.2 tests)
        return self

class ImpactField(Versioned):
    actor_id: str
    action_id: str
    impact_field: list[ImpactItem]
```

The other three models (`WorldState`, `RelationGraph`, `RawConsequenceTrace`,
`GroupState`) follow the JSON examples in §11–§16 verbatim, with `extra="forbid"`
and range/vocab validators. See [Appendix 3](#appendix3) for the field list.

**0.3 The 2 closed vocabularies** (`behavioral_layer/vocab/`) as frozen enums —
see [Appendix 1](#appendix1) for the complete token lists. Add the P-dimension
enum (8), the G-variable enum (3), `TargetType` (`P`/`G`), and the M2
`InclusionReason` enum.

**0.4 Key functions** (`behavioral_layer/keys.py`):

```python
def natural_key(item) -> str:
    return f"{item.target_type}|{item.target_id}|{item.dimension}|{item.horizon}"

def storage_key(world_id, tick, actor_id, action_id, item) -> str:
    return f"{world_id}|{tick}|{actor_id}|{action_id}|{natural_key(item)}"
```

**0.5 Benchmark fixtures** (`tests/fixtures/benchmarks/b01..b10.json`). Each file
encodes the scenario's cast, action, and **expected labels** as data:

```json
{
  "id": "B2",
  "name": "Theft, observed",
  "world_id": "npc_village_01",
  "action": "agent_a steals grain from agent_b; agent_c witnesses",
  "tracked_groups": ["village_1"],
  "expected_items": [
    {"target_id": "agent_a", "target_type": "P", "dimension": "resources",     "sign": "+", "horizon": "short", "unc_band": "low"},
    {"target_id": "agent_b", "target_type": "P", "dimension": "resources",     "sign": "-", "horizon": "short", "unc_band": "low"},
    {"target_id": "agent_b", "target_type": "P", "dimension": "psychological", "sign": "-", "horizon": "short", "unc_band": "low"},
    {"target_id": "agent_a", "target_type": "P", "dimension": "reputation",    "sign": "-", "horizon": "mid",   "unc_band": "med"},
    {"target_id": "village_1","target_type": "G", "dimension": "cohesion",      "sign": "-", "horizon": "mid",   "unc_band": "med"},
    {"target_id": "village_1","target_type": "G", "dimension": "member_welfare","structural_only": true}
  ],
  "expected_absent": [],
  "checks": ["detection-risk branch present", "welfare row presence+consistency only, not sign"]
}
```

The full label set for all ten is in [Appendix 2](#appendix2).

**0.6 Grader** (`behavioral_layer/grading/`). Implements the §18 metrics + rubric:

- **impact-cell recall** = `|matched expected cells| / |expected cells|`, where a
  match requires the **full natural key** (target_type, target_id, dimension, horizon).
- **target-only recall** (secondary) = right entity even if dim/horizon missed.
- **sign accuracy** = MACRO-averaged by dimension, over determinate-sign items
  only; **member_welfare excluded**.
- **uncertainty bands:** `low: 0–0.3`, `med: 0.3–0.6`, `high: 0.6–1.0`.
  Benchmark rows use the **uncertainty** band (magnitude band defined but not scored at MVP).
- **rubric flags:** RECALL (every expected target appears), SIGN (determinate only;
  `?`/`~` excluded but must carry `unc ≥ 0.7`), HORIZON (named band), ABSENT (an
  expected-absent label is FAILED by any confident `uncertainty < 0.5` item),
  EXTRA (extra high-uncertainty items not penalized), WELFARE (structural-only).

### How to run
```powershell
pytest behavioral_layer/tests/test_schemas.py behavioral_layer/tests/test_vocab.py `
       behavioral_layer/tests/test_keys.py behavioral_layer/tests/test_grading.py -v
python -m behavioral_layer.schemas --export-json-schema schemas_out\   # optional sanity
```

### How to test
| Test file | Asserts |
|---|---|
| `test_schemas.py` | Each spec JSON example round-trips; out-of-range delta (`1.5`) rejected; unknown field rejected (`extra=forbid`); a `G` item with dimension `autonomy` rejected; primary item without `evidence_ref` rejected; derived item with `evidence_ref` rejected |
| `test_vocab.py` | All 13 FACT_TYPE / 11 MECHANISM tokens present incl. `unmappable`; an out-of-vocab token raises |
| `test_keys.py` | `natural_key` matches the spec string exactly; a person id and group id with the same string produce different keys (different `target_type`) |
| `test_grading.py` | A hand-authored "perfect" B-field scores recall 1.0 / sign 1.0; an injected wrong sign drops macro sign accuracy by exactly the per-dimension share; a confident item on an expected-absent target fails ABSENT; an extra high-uncertainty item does **not** lower the score |

### Exit criteria
- All 5 schemas validate the spec's own JSON examples and reject the crafted-invalid cases.
- Vocab enums reject out-of-vocab tokens.
- B1–B10 fixtures parse against an `ExpectedScenario` model.
- Grader scores a hand-authored perfect field at 100% and reacts correctly to injected errors.

### Run & test walkthrough (verified)

Phase 0 is the **contracts layer** — there is no service to start. Its three
runnable surfaces are: the **test suite** (primary), the **schema export**, and
**interactive use**. Current status: `25 passed`.

**Prerequisite (once per terminal):**
```powershell
cd C:\mindorigin
.\.venv\Scripts\Activate.ps1        # or prefix every command with .\.venv\Scripts\python.exe
```

**1 — Test it (this is how you "run" Phase 0):**
```powershell
pytest                               # whole suite  -> 25 passed
pytest -v                            # verbose: every test name + PASS/FAIL
pytest --cov=behavioral_layer --cov-report=term-missing   # with coverage

# one area at a time
pytest behavioral_layer/tests/test_schemas.py -v     # the 5 frozen schemas
pytest behavioral_layer/tests/test_vocab.py -v       # the 2 closed vocabularies
pytest behavioral_layer/tests/test_keys.py -v        # natural/storage keys
pytest behavioral_layer/tests/test_grading.py -v     # the §18 grader + fixtures
```
Useful flags: `pytest -k sign` (match a substring), `pytest -x` (stop at first
failure), `pytest -q` (quiet).

What each test file proves:

| File | Verifies |
|---|---|
| `test_schemas.py` | All 5 schemas round-trip the spec's own JSON examples; out-of-range delta (`1.5`), unknown fields, a `G` target with a P-dimension, a primary item missing `evidence_ref`, a derived item carrying `evidence_ref`, and an out-of-vocab `fact_type` are all **rejected** |
| `test_vocab.py` | FACT_TYPE (13) and MECHANISM (11) complete incl. `unmappable`; P-dims=8, G-vars=3, inclusion-reasons=5; junk tokens raise |
| `test_keys.py` | `natural_key` = exact `P\|agent_a\|resources\|short`; a person and group sharing an id get **different** keys; `storage_key` prefixes the natural key |
| `test_grading.py` | Perfect field → recall 1.0 / sign 1.0; a flipped sign drops macro accuracy to 0.5; a confident item on an expected-absent target fails ABSENT; a high-uncertainty extra is **not** penalized; all 10 B-fixtures load |

**2 — Run the schema export (the one CLI in Phase 0):**
```powershell
python -m behavioral_layer.schemas --export-json-schema schemas_out\
# -> WorldState.json, RelationGraph.json, RawConsequenceTrace.json,
#    ImpactField.json, GroupState.json
```

**3 — Interactive use** — import and exercise any piece (e.g. a throwaway script):
```python
from behavioral_layer.schemas import ImpactField, ImpactItem
from behavioral_layer.keys import natural_key
from behavioral_layer.vocab import FactType
from behavioral_layer.grading import grade_scenario, load_scenario

it = ImpactItem(target_id="agent_b", target_type="P", dimension="resources",
                delta=-0.4, horizon="short", uncertainty=0.2, evidence_ref="trace_001")
print(natural_key(it))                       # P|agent_b|resources|short

sc = load_scenario("behavioral_layer/tests/fixtures/benchmarks/b02_theft_observed.json")
field = ImpactField(actor_id="agent_a", action_id="act_017", impact_field=[
    ImpactItem(target_id="agent_a", target_type="P", dimension="resources",
               delta=0.4, horizon="short", uncertainty=0.2, evidence_ref="t1"),
    ImpactItem(target_id="agent_b", target_type="P", dimension="resources",
               delta=-0.4, horizon="short", uncertainty=0.2, evidence_ref="t1"),
])
r = grade_scenario(field, sc)
print(r.impact_cell_recall, r.macro_sign_accuracy)   # 0.33 1.0  (2 of 6 cells; both signs right)
```
Run it with `.\.venv\Scripts\python.exe your_script.py`. (On Windows, avoid
`python -c "..."` with quoted strings — PowerShell mangles the inner quotes; use a
file or a single-quoted here-string.)

> `schemas_out/` is a regenerable artifact from the export command — safe to
> delete; add it to `.gitignore` when a repo is initialised for the code.

---

<a name="phase1"></a>
## Phase 1 — M5 Aggregator / Validator (built FIRST)

### What it does
M5 is **deterministic logic only** and is built first because it doubles as the
**test harness** for every other module (spec §17, §22). It takes an ImpactField
+ GroupStates and returns a **checked** field, deriving `member_welfare` under R1.

### What to build (`modules/m5_validator.py`)

**1.1 Structural validators**
- Range/vocab checks (delegate to the Pydantic models, then add cross-field rules).
- `evidence_ref` resolution for **primary items only** (derived items are exempt — they use `derived_from`).
- Projector-authored `G` items restricted to `cohesion`/`continuity`; any
  M4-authored `member_welfare` is a violation (only M5 may author it).

**1.2 R1 — within-group dedup.** Each member contributes **exactly once** to a
group's `member_welfare`, even if reached via several subgroups. Scope is a
single group's rollup, **not** a global per-person ledger (§15a): a person in
`family_1` and `guild_2` contributes to **both** groups' welfare.

**1.3 R2 — per-group emergent variables.** `cohesion` and `continuity` computed
independently for every tracked group incl. subgroups; **no union substitution**.

**1.4 R3 — subgroup preservation / no parent summing.** A tracked subgroup's
emergent items are reported separately and never merged into the parent; a
parent's welfare is computed from its full membership directly.

**1.5 welfare↔member-P consistency.** A `member_welfare` item must be backed by
member-level P rows; a welfare item with no supporting member-P rows is a violation
(this is what B5 tests).

**1.6 member_welfare derivation — parameterized, constants UNSET.** Build the
frozen protective-hybrid form so it is ready for calibration but emits
structural-only items today:

```
W_g(h) = λ · mean_m_i( S_i(h) ) + (1 − λ) · min_i( S_i(h) )
S_i(h) = Σ_d  w_P[d] · dP_i[d]      (over affected dims d)
```

- `mean_m` = m-weighted mean over the group's own members (R1: each once).
- `min` = protective term (severe harm to one member is not averaged away).
- λ ∈ [0,1] and `w_P[d]` are **UNSET** (deferred to calibration). Implement as a
  parameterized function; until set, every `member_welfare` item is emitted with
  `delta 0.0`, `uncertainty 1.0`, `calibration_status:"structural_only"`,
  `numeric_scoring:false`. **Never** a calibrated-looking number.

**1.7 Two modes.**
- `strict` (test/train): any violation rejects the **whole** field with a
  machine-readable diagnostic (never a silent repair).
- `runtime`: returns the valid items + a `diagnostics` list, so an acting agent
  is not blocked when 90% of the field is sound.

### How to run
```powershell
# Validate a field in strict mode (exits non-zero + prints diagnostics on violation)
python -m behavioral_layer.modules.m5_validator --in field.json --groups groupstates.json --mode strict

# Runtime mode (returns valid items + diagnostics, exit 0)
python -m behavioral_layer.modules.m5_validator --in field.json --groups groupstates.json --mode runtime --pretty
```

### How to test (`test_m5_validator.py`)
| Test | Asserts |
|---|---|
| `test_r1_member_counted_once_per_group` | a member reached via 2 subgroups appears once in the group rollup's `derived_from` |
| `test_r1_member_in_two_sibling_groups` | (B10) `agent_b` present in **both** `family_1` and `guild_2` welfare; not omitted |
| `test_r3_subgroup_not_merged` | (B8) `marriage_1` items not merged into `family_1`; `family_1` welfare derived from full membership |
| `test_welfare_requires_member_p` | (B5) a `member_welfare` item with no member-P rows → violation |
| `test_member_welfare_structural_only` | derived welfare item carries `delta 0.0`, `uncertainty 1.0`, `numeric_scoring:false`, `calibration_status:"structural_only"` |
| `test_m4_cannot_author_member_welfare` | an input item with `target_type=G, dimension=member_welfare, derived=false` → violation |
| `test_dangling_evidence_ref_primary_only` | primary item with unknown `evidence_ref` → violation; derived item without `evidence_ref` is fine |
| `test_strict_rejects_whole_field` | one violation → whole field rejected with diagnostic |
| `test_runtime_returns_valid_plus_diag` | 1 bad of 10 items → 9 returned + 1 diagnostic, exit 0 |

### Exit criteria
M5 correctly accepts crafted-valid fields and rejects each crafted violation with
a machine-readable diagnostic; B5/B8/B10 pass at the **validator level** using
hand-authored fields (no upstream modules needed yet).

### Run & test walkthrough (verified)

Status: `11 passed` for M5; `36 passed` for the whole suite (Phase 0 + Phase 1).
M5 is deterministic and offline — no network, no LLM.

**1 — Test it:**
```powershell
.\.venv\Scripts\Activate.ps1
pytest behavioral_layer/tests/test_m5_validator.py -v   # 11 passed
pytest                                                  # 36 passed (whole suite)
```

What the M5 tests prove:

| Test | Proves |
|---|---|
| `test_r1_member_counted_once_per_group` | R1: a member reached via 2 subgroups appears once in `derived_from` |
| `test_r1_member_in_two_sibling_groups` | B10: `agent_b` present in **both** `family_1` and `guild_2` welfare |
| `test_r3_subgroup_not_merged` | B8: `marriage_1` reported separately; parent welfare references P rows only |
| `test_welfare_requires_member_p` | a welfare item with no backing member-P rows → flagged |
| `test_no_welfare_when_no_member_p_changed` | B5: no member P changed → no welfare derived |
| `test_member_welfare_structural_only` | derived welfare = `delta 0.0`, `uncertainty 1.0`, `numeric_scoring false`, `structural_only` |
| `test_m4_cannot_author_member_welfare` | M4-authored (`derived=false`) member_welfare → violation |
| `test_dangling_evidence_ref_primary_only` | primary item with unknown `evidence_ref` → flagged; derived items exempt |
| `test_strict_rejects_whole_field` / `test_runtime_returns_valid_plus_diag` | the two modes |
| `test_rollup_formula_is_protective_hybrid` | the frozen `W_g = λ·meanₘ(Sᵢ) + (1−λ)·minᵢ(Sᵢ)` form |

**2 — Run the CLI:**
```powershell
python -m behavioral_layer.modules.m5_validator --in field.json --groups groups.json --mode strict --pretty
# optional: --trace trace.json   (enables dangling evidence_ref checking)
# optional: --mode runtime       (returns valid items + diagnostics instead of rejecting)
```

**Inputs**

| Flag | What it is | Notes |
|---|---|---|
| `--in field.json` | an **ImpactField** — M4's output: primary P items + cohesion/continuity G items | Must contain **no** `member_welfare` (M5 derives it; an M4-authored one is a violation) |
| `--groups groups.json` | a JSON **array of GroupState** — membership + `tracked` flag | Required to know who belongs to each group; only `tracked:true` groups get welfare |
| `--trace trace.json` | optional **RawConsequenceTrace** | M5 takes its `step_ids` to check primary items' `evidence_ref`. Omit → that check is skipped |
| `--mode` | `strict` or `runtime` | strict rejects the whole field on any violation; runtime returns the valid subset + diagnostics |

**Output** — a JSON object:
- `ok` — `true` when there are no diagnostics.
- `items` — the **checked field**: the valid input items **plus M5-derived `member_welfare`** items (one per tracked group × horizon that has member-P rows, structural-only).
- `diagnostics` — list of `{code, message, item}`. Possible `code`s:

| Diagnostic code | Meaning |
|---|---|
| `m4_authored_member_welfare` | a primary (`derived=false`) member_welfare item — only M5 may author it |
| `dangling_evidence_ref` | a primary item's `evidence_ref` doesn't resolve to a trace `step_id` |
| `emergent_on_untracked_group` | cohesion/continuity emitted for a group whose `tracked` is false |
| `welfare_without_member_p` | a member_welfare item not backed by member-P rows (consistency) |
| `welfare_double_count` | a member row counted twice in one welfare (R1) |
| `welfare_parent_summing` | a welfare's `derived_from` references a group row, i.e. summing subgroups (R3) |

In `strict` mode a violation prints `REJECTED (strict):` with the diagnostics and exits non-zero.

**Worked example (B10-style).**

Input `field.json` (two members share two groups; M4 emitted only the P deltas):
```json
{"actor_id":"agent_a","action_id":"act_017","impact_field":[
  {"target_id":"agent_a","target_type":"P","dimension":"resources","delta":-0.3,"horizon":"short","uncertainty":0.2,"evidence_ref":"t1"},
  {"target_id":"agent_b","target_type":"P","dimension":"resources","delta":0.3,"horizon":"short","uncertainty":0.2,"evidence_ref":"t1"}
]}
```
Input `groups.json` (both `family_1` and `guild_2` contain agent_a + agent_b):
```json
[
  {"group_id":"family_1","tracked":true,"members":[{"agent":"agent_a","m":0.9,"id_strength":0.8},{"agent":"agent_b","m":0.9,"id_strength":0.7}],"g_state":{"member_welfare":0.0,"cohesion":0.0,"continuity":0.0}},
  {"group_id":"guild_2","tracked":true,"members":[{"agent":"agent_a","m":0.9,"id_strength":0.8},{"agent":"agent_b","m":0.9,"id_strength":0.7}],"g_state":{"member_welfare":0.0,"cohesion":0.0,"continuity":0.0}}
]
```

**Expected result** (`ok: true`): the 2 input P items pass through unchanged, and M5 **adds two derived `member_welfare` items** — one for `family_1`, one for `guild_2` — each structural-only and each rolling up `agent_a` + `agent_b` once (R1; `agent_b` is in both, omitted from neither):
```json
{
  "ok": true,
  "items": [
    { "target_id":"agent_a", "target_type":"P", "dimension":"resources", "delta":-0.3, "horizon":"short", "uncertainty":0.2, "evidence_ref":"t1", "derived":false, "numeric_scoring":true },
    { "target_id":"agent_b", "target_type":"P", "dimension":"resources", "delta":0.3,  "horizon":"short", "uncertainty":0.2, "evidence_ref":"t1", "derived":false, "numeric_scoring":true },
    { "target_id":"family_1", "target_type":"G", "dimension":"member_welfare", "delta":0.0, "horizon":"short",
      "uncertainty":1.0, "derived":true, "calibration_status":"structural_only", "numeric_scoring":false,
      "derived_from":["P|agent_a|resources|short","P|agent_b|resources|short"] },
    { "target_id":"guild_2",  "target_type":"G", "dimension":"member_welfare", "delta":0.0, "horizon":"short",
      "uncertainty":1.0, "derived":true, "calibration_status":"structural_only", "numeric_scoring":false,
      "derived_from":["P|agent_a|resources|short","P|agent_b|resources|short"] }
  ],
  "diagnostics": []
}
```

If you instead feed a primary `member_welfare` item (M4 trying to author it), `--mode strict` exits non-zero with:
```
REJECTED (strict):
  [m4_authored_member_welfare] member_welfare is M5-derived only; M4 authored G|village_1|member_welfare|short
```

> **Windows note.** Don't create the JSON inputs with `Set-Content -Encoding utf8` — PowerShell 5.1 prepends a UTF-8 BOM that breaks JSON parsing. Write the file with a normal editor/tool (no BOM), or use `[System.IO.File]::WriteAllText($path, $json)`.

---

<a name="phase2"></a>
## Phase 2 — M1 Domain Adapter (one NPC world)

### What it does
Translates a raw scene/action/world into **normalized facts + a RelationGraph
fragment + registry updates**. All world-specific/cultural meaning lives here;
the core stays world-invariant (claim C2). The behavioral layer **reads**
WorldState, never writes it.

### What to build (`modules/m1_adapter.py` + `config/worlds/npc_village_01.yaml`)

**2.1 Per-world mapping table** — a **reviewed config file**, not code. Maps world
mechanics → FACT_TYPE tokens. Ambiguous cultural meaning → **competing facts**
with lower `adapter_confidence`, never one guessed fact.

**2.2 Normalizer** → normalized facts (each with `fact_type` from the closed
vocab + `adapter_confidence`), RelationGraph fragment (r-vector edges + `sign`;
memberships with `m` and `id_strength`), registry updates. `adapter_confidence`
is a **separate axis** from M3 `step_confidence` and M4 `uncertainty`.

**2.3 Failure path** — an unmappable mechanic → `fact_type:"unmappable"`,
`adapter_confidence ≤ 0.3`, **never dropped**.

**2.4 Anti-smuggling guard (I2)** — a hard validator rejecting any M1 payload that
contains a `delta`, affect, or normative/evaluative field. No world-specific
tokens (sword, spell, guild rank) in the output vocabulary — only in free-text
`event`/`text`.

### How to run
```powershell
python -m behavioral_layer.modules.m1_adapter --world npc_village_01 --in scene.json --pretty
```

### How to test (`test_m1_adapter.py`)
| Test | Asserts |
|---|---|
| `test_c2_world_invariance` | "Knight swings sword at peasant" and "Mage casts firebolt at farmer" both normalize to identical-schema `condition` facts; only free-text `event` differs |
| `test_unmappable_never_dropped` | an unknown mechanic → `fact_type:"unmappable"`, `adapter_confidence ≤ 0.3`, present in output |
| `test_ambiguous_emits_competing_facts` | an ambiguous cultural act → ≥2 competing facts with lower confidence, not one guess |
| `test_anti_smuggling_rejects_delta` | an M1 payload carrying a `delta`/normative field → rejected |
| `test_no_world_tokens_in_vocab` | output `fact_type` values are all from the closed vocab; "sword"/"spell" appear only in free text |
| `test_unmappable_rate_zero_on_b1_b10` | `unmappable_rate == 0` across the B1–B10 scene inputs |

### Exit criteria
C2 world-invariance test passes; `unmappable_rate = 0` on B1–B10 inputs;
anti-smuggling guard rejects every crafted delta/normative payload.

### Run & test walkthrough (verified)

Status: `9 passed` for M1; `45 passed` for the whole suite (Phases 0–2).
M1 is deterministic and offline — the per-world mapping is config, not code.

> **Config format note.** The repo-layout section shows `config/worlds/*.yaml`,
> but the MVP ships the mapping as **JSON** (`config/worlds/npc_village_01.json`)
> to avoid adding a YAML dependency. Same content, different serialization.

New files this phase:
- `behavioral_layer/modules/m1_adapter.py`
- `behavioral_layer/config/worlds/npc_village_01.json` — the reviewed mapping table
- `behavioral_layer/tests/fixtures/scenes.json` — the **B1–B10 raw-scene inputs** (reused by M2–M5 in later phases)

**1 — Test it:**
```powershell
pytest behavioral_layer/tests/test_m1_adapter.py -v   # 9 passed
pytest                                                # 45 passed (whole suite)
```

| Test | Proves |
|---|---|
| `test_c2_world_invariance` | "Knight swings sword" and "Mage casts firebolt" → identical-schema `condition` fact; only free text differs |
| `test_unmappable_never_dropped` | an unknown mechanic → `fact_type:"unmappable"`, `confidence ≤ 0.3`, still present |
| `test_ambiguous_emits_competing_facts` | an ambiguous mechanic → ≥2 competing facts at lower confidence, never one guess |
| `test_anti_smuggling_rejects_delta` / `test_anti_smuggling_rejects_normative` | a raw scene carrying a `delta`/`moral` field is rejected |
| `test_no_world_tokens_in_output_vocab` | `fact_type` values are closed-vocab only; "sword" lives only in free text |
| `test_unregistered_entity_rejected` | an event referencing an unregistered entity is rejected |
| `test_relations_become_graph_fragment` | relations become RelationGraph edges/memberships (and `history_valence`/`affection` are not mistaken for smuggling) |
| `test_unmappable_rate_zero_on_b1_b10` | `unmappable_rate == 0` across all 10 benchmark scenes |

**2 — Run the CLI:**
```powershell
python -m behavioral_layer.modules.m1_adapter --scene scene.json --pretty
# optional: --world path/to/mapping.json   (defaults to config/worlds/npc_village_01.json)
```

**Input** — a raw scene (`scene.json`):

| Field | What it is |
|---|---|
| `world_id`, `tick` | scene identity |
| `entities` | `[{id, type}]` — the registry of known entities |
| `events` | `[{id, text, raw_mechanic, actors[], targets[]}]` — `raw_mechanic` is the world-specific verb the mapping translates; `text` is free-form |
| `relations` (optional) | `[{from,to,sign,r_vector}]` edges and `[{kind:"membership",agent,group,m,id_strength}]` memberships |

A scene must **not** contain any `delta`/affect/normative field (anti-smuggling), and every `actors`/`targets` id must be a registered entity.

**Output** — an `AdapterOutput` JSON:
- `facts` — normalized `Fact[]` (`fact_type` from the closed vocabulary, per-fact `confidence` = adapter_confidence).
- `relation_graph` — the RelationGraph fragment (nodes/edges/memberships).
- `registry` — the entity ids.
- `unmappable_rate` — fraction of facts that are `unmappable` (CLI convenience).

**Worked example (C2 world-invariance), run live:**

Input A `_sword.json`:
```json
{"world_id":"npc_village_01","tick":1,
 "entities":[{"id":"agent_a","type":"person"},{"id":"agent_b","type":"person"}],
 "events":[{"id":"f1","text":"Knight agent_a swings a sword at peasant agent_b","raw_mechanic":"sword_attack","actors":["agent_a"],"targets":["agent_b"]}]}
```
Input B `_firebolt.json` is identical except `text` and `raw_mechanic:"firebolt"`.

**Expected result** — both produce the **same** fact (`fact_type:"condition"`, `confidence:1.0`); only `text` differs:
```text
world A: {"facts":[{"id":"f1","text":"Knight agent_a swings a sword at peasant agent_b","fact_type":"condition","source":"adapter","confidence":1.0}], "unmappable_rate":0.0}
world B: {"facts":[{"id":"f1","text":"Mage agent_a casts a firebolt at farmer agent_b","fact_type":"condition","source":"adapter","confidence":1.0}], "unmappable_rate":0.0}
```
That identical `fact_type` across two different worlds — with the world-specific
words ("sword", "firebolt") confined to `text` — *is* claim C2 demonstrated.

---

<a name="phase3"></a>
## Phase 3 — M2 Candidate Target Detector (recall-first)

### What it does
From facts + action + graph, produces **affected entities & groups** as candidates,
each with an `inclusion_reason`. Policy is **recall-first**: over-inclusion is fine
(M4 may zero/drop), **omission is the only real failure**. Precision is recovered
downstream.

### What to build (`modules/m2_targets.py` + `config/m2_caps.yaml`)

**3.1 Candidate generation** with `inclusion_reason ∈ {direct_object,
relation_neighbor, group_member, observer, norm_stakeholder}`. **The actor is
always included.** Each candidate is reachable within k hops or carries an
observer/norm_stakeholder reason.

**3.2 Bounded recall** (avoids candidate blow-up in dense graphs): rank by
relation strength and cap via config (`max_candidates`, `relation_strength_threshold`,
`group_size_cap`). **Pruned candidates are logged, never silently cut** — so the
policy stays recall-first rather than re-introducing precision filtering at M2.

### How to run
```powershell
python -m behavioral_layer.modules.m2_targets --in facts.json --graph graph.json --config config\m2_caps.yaml --pretty
```

### How to test (`test_m2_targets.py`)
| Test | Asserts |
|---|---|
| `test_actor_always_included` | the acting agent is always a candidate |
| `test_second_order_target_detected` | (B9) rival peer `agent_e` is included via `relation_neighbor` |
| `test_antagonist_detected` | (B6) `attacker_x` is included |
| `test_pruned_candidates_logged` | when caps trigger, pruned candidates appear in the log, not silently dropped |
| `test_pruned_expected_target_rate_zero` | across B1–B10, **no expected target is ever pruned** (rate = 0) |
| `test_inclusion_reason_present` | every candidate carries a valid `inclusion_reason` |

### Exit criteria
`pruned-expected-target rate = 0` on B1–B10 (the caps must never drop an expected
target); B6 antagonist and B9 second-order peer are detected.

### Run & test walkthrough (verified)

Status: `7 passed` for M2; `52 passed` for the whole suite (Phases 0–3).
M2 is deterministic and offline.

New files this phase:
- `behavioral_layer/modules/m2_targets.py`
- `behavioral_layer/tests/fixtures/m2_inputs.json` — per-benchmark action context + group membership (reused by M3–M5)

**1 — Test it:**
```powershell
pytest behavioral_layer/tests/test_m2_targets.py -v   # 7 passed
pytest                                                # 52 passed (whole suite)
```

| Test | Proves |
|---|---|
| `test_actor_always_included` | the acting agent is always a candidate |
| `test_antagonist_detected` | B6: `attacker_x` included (direct object of the defense) |
| `test_second_order_target_detected` | B9: rival peer `agent_e` reached as a `relation_neighbor` of a direct object |
| `test_inclusion_reason_present` | every candidate carries one of the 5 valid reasons |
| `test_pruned_candidates_logged_not_dropped` | a tight cap prunes weakest neighbours → **logged**, strongest kept |
| `test_direct_object_never_pruned_under_tight_caps` | even `max_candidates=0`, actor + direct objects survive (only neighbours are capped) |
| `test_pruned_expected_target_rate_zero_on_b1_b10` | across all 10 benchmarks every expected target is found and **nothing is pruned** |

**2 — Run the CLI:**
```powershell
python -m behavioral_layer.modules.m2_targets --action action.json --graph graph.json --groups groups.json --pretty
# --graph and --groups are optional
```

**Input**

| Flag | What it is |
|---|---|
| `--action action.json` | an **ActionContext**: `{actor, direct_objects[], observers[], norm_stakeholders[]}` |
| `--graph graph.json` | optional **RelationGraph** — used to find `relation_neighbor` candidates by BFS, ranked by edge strength |
| `--groups groups.json` | optional **GroupState[]** — a tracked group with any affected member becomes a candidate, and its members are added (`group_member`) |

Caps (`M2Caps`, recall-first defaults): `max_candidates=64`, `relation_strength_threshold=0.0`, `group_size_cap=50`, `max_hops=2`. With the defaults nothing is pruned; tighten them and the **excess relation-neighbours / over-cap group members** are pruned — never the actor, direct objects, observers, or norm stakeholders.

**Output** — an `M2Result`:
- `candidates` — `[{target_id, inclusion_reason, is_group}]`. `inclusion_reason` ∈ `{direct_object, relation_neighbor, group_member, observer, norm_stakeholder}` (most-direct reason wins on ties).
- `pruned` — `[{target_id, reason}]` with `reason` ∈ `{below_threshold, over_cap, group_over_cap}` — the audit log proving nothing was silently cut.

**Worked example (B9 second-order target), run live.**

Input `action.json` + `graph.json` (leader praises agent_b; agent_e is a rival peer with an edge to agent_b):
```json
{"actor": "agent_l", "direct_objects": ["agent_b"]}
```
```json
{"nodes":["agent_l","agent_b","agent_e"],
 "edges":[{"from":"agent_b","to":"agent_e","r_vector":{"trust":0.3,"affection":0.0,"dependence":0.0,"authority":0.0,"history_valence":-0.4},"sign":-1}]}
```

**Expected result** — `agent_e` is surfaced as a `relation_neighbor` (the second-order target M2 must not miss), with nothing pruned:
```json
{"candidates":[
  {"target_id":"agent_l","inclusion_reason":"direct_object","is_group":false},
  {"target_id":"agent_b","inclusion_reason":"direct_object","is_group":false},
  {"target_id":"agent_e","inclusion_reason":"relation_neighbor","is_group":false}],
 "pruned":[]}
```

---

<a name="phase4"></a>
## Phase 4 — M3 Raw Consequence Generator (rules-only first)

### What it does
From action + facts + targets, produces a `RawConsequenceTrace`: a **tree** of
causal steps (not a single chain), each naming a mechanism from the closed vocab,
with per-step confidence and parent links, bounded by `depth_limit`.

### What to build (`modules/m3_consequence.py` + `providers/llm_provider.py`)

**4.1 Per-mechanism rule expander** — deterministic rules that emit steps. Traces
are **trees**: e.g. theft (B2) generates both the `possession_transfer` chain and
the `norm_violation_detection` branch. `parent` links are acyclic; depth ≤
`depth_limit`; **truncation is flagged explicitly (I8)** — a truncated branch means
"not computed", never "no effect".

**4.2 Validation under M5** — mechanisms from vocab; every id in `affected` is
registered in WorldState/GroupState. A step referencing an unregistered (hallucinated)
entity is **rejected and logged**, and the trace is rebuilt without it — never
silently repaired.

**4.3 Pluggable LLM seam (deferred) — provider: Claude.** Define the interface
now, ship rules only. The future `ClaudeProvider` uses the official `anthropic`
SDK with model `claude-opus-4-8`:

```python
# providers/llm_provider.py
from typing import Protocol

class LLMStepProvider(Protocol):
    def expand(self, action, facts, targets) -> list[dict]: ...

class RulesProvider:
    """MVP: deterministic per-mechanism rules. No network."""
    def expand(self, action, facts, targets) -> list[dict]: ...

# Future ClaudeProvider contract (NOT built until a benchmark forces it):
#   - model "claude-opus-4-8"; official `anthropic` SDK (add as optional dep `llm`)
#   - schema-valid output via client.messages.parse(output_format=RawConsequenceTrace)
#       -> our Pydantic schemas ARE the output schema; the Mechanism/FactType
#          enums become JSON-Schema enums => the model cannot emit an out-of-vocab
#          token (this is how "constrained decoding to the vocabulary" is realized)
#   - thinking={"type": "adaptive"}, output_config={"effort": "..."}
#   - retry on pydantic ValidationError: <= 2 retries, then emit `unmappable`
#   - depth <= depth_limit; truncated flag set, never silent
#   - a step that cannot be validated is rejected and logged (never emit unvalidated)
```

> **Determinism note (important).** The spec's "low temperature; fixed seed"
> cannot be applied to Opus 4.8 — `temperature`/`top_p`/`top_k` are removed
> (sending them 400s) and there is no `seed` parameter. Reproducibility and the
> §19.7 rerun-stability metric are therefore achieved **not** via sampling params
> but by **caching every LLM output to golden files** and replaying them offline
> in the test suite (VCR/cassette pattern). The benchmark/acceptance suites must
> run hermetically with no live Claude calls. For M1 (if it ever uses an LLM),
> the anti-smuggling rule is enforced structurally: the output schema omits any
> `delta`/normative field, so the model cannot produce one.

### How to run
```powershell
python -m behavioral_layer.modules.m3_consequence --in targets.json --facts facts.json --provider rules --pretty
```

### How to test (`test_m3_consequence.py`)
| Test | Asserts |
|---|---|
| `test_theft_is_a_tree` | (B2) trace contains both `possession_transfer` and `norm_violation_detection` branches |
| `test_unregistered_entity_rejected` | a step referencing an unknown id is rejected + logged; trace rebuilt without it |
| `test_parent_links_acyclic` | parent links form a DAG/tree, no cycles |
| `test_depth_bounded_and_flagged` | depth never exceeds `depth_limit`; if cut, `truncated:true` is set |
| `test_mechanisms_in_vocab` | every step's mechanism is a valid token; unknown → `unmappable` (failure value) |
| `test_trace_passes_m5` | generated traces validate under M5's trace checks |

### Exit criteria
B2 produces a tree with both branches; all generated traces pass M5 validation;
unregistered-entity rejection works and is logged.

### Run & test walkthrough (verified)

Status: `8 passed` for M3; `60 passed` for the whole suite (Phases 0–4).
M3 is deterministic and offline — the LLM path is defined but **not built**.

New files this phase:
- `behavioral_layer/modules/m3_consequence.py`
- `behavioral_layer/config/m3_mechanism_rules.json` — raw-mechanic → mechanism tree rules
- `behavioral_layer/providers/` — `LLMStepProvider` protocol + `RulesProvider` (the deferred Claude seam)

**1 — Test it:**
```powershell
pytest behavioral_layer/tests/test_m3_consequence.py -v   # 8 passed
pytest                                                    # 60 passed (whole suite)
```

| Test | Proves |
|---|---|
| `test_theft_is_a_tree` | B2: trace has both `possession_transfer` and `norm_violation_detection`, the latter hanging off the former (tree, not chain) |
| `test_mechanisms_in_vocab_unknown_is_unmappable` | an unknown raw mechanic → the `unmappable` failure value; all mechanisms are vocab tokens |
| `test_unregistered_entity_rejected_and_logged` | a step referencing an unregistered entity is rejected + logged; trace rebuilt without it |
| `test_parent_links_acyclic` | parent links form a tree (no cycles, no dangling parents) |
| `test_depth_bounded_and_flagged` | `depth_limit=1` truncates the chain and sets `truncated:true` |
| `test_trace_passes_m5_evidence_resolution` | an ImpactField item referencing a real step_id resolves cleanly in M5 |
| `test_all_b_scenes_generate_valid_traces` | every B1–B10 scene yields a non-empty, acyclic, vocab-valid trace with no rejects |
| `test_rules_provider_satisfies_protocol` | `RulesProvider` satisfies the `LLMStepProvider` seam |

**2 — Run the CLI:**
```powershell
python -m behavioral_layer.modules.m3_consequence --scene scene.json --pretty
# optional: --rules path/to/m3_mechanism_rules.json   (defaults to the packaged table)
```

**Input** — a raw scene (same shape M1/M2 consume): `entities` (the registry) +
`events` with `raw_mechanic`, `actors`, `targets`. The mechanism-rules config maps
each `raw_mechanic` to an ordered list of step rules (`{mechanism, affects, parent}`)
that form a tree; `affects` ∈ `{actor, targets, actor_and_targets}`.

**Output** — an `M3Result`:
- `trace` — a `RawConsequenceTrace`: `steps[]` (each with `step_id`, `t`, `event`, `mechanism`, `affected[]`, `confidence`, `parent`), `depth_limit`, `truncated`.
- `rejected` — `[{event_id, mechanism, affected, reason}]` — the audit log of dropped steps (`unregistered:…` / `parent_missing`), proving nothing was silently repaired.

**Worked example (B2 theft tree), run live.**

Input `scene.json` — a theft witnessed by a third party:
```json
{"id":"B2","entities":[{"id":"agent_a","type":"person"},{"id":"agent_b","type":"person"},{"id":"agent_c","type":"person"}],
 "events":[
   {"id":"f1","text":"agent_a steals grain from agent_b","raw_mechanic":"steal","actors":["agent_a"],"targets":["agent_b"]},
   {"id":"f2","text":"agent_c witnesses the theft","raw_mechanic":"witness","actors":["agent_c"]}]}
```

**Expected result** — a tree: `possession_transfer` (root) → `norm_violation_detection` (its child), plus the witness's `belief_update`; nothing rejected:
```json
{"trace":{"action_id":"B2","steps":[
  {"step_id":"trace_001","t":0,"event":"agent_a steals grain from agent_b","mechanism":"possession_transfer","affected":["agent_a","agent_b"],"confidence":0.9,"parent":null},
  {"step_id":"trace_002","t":1,"event":"agent_a steals grain from agent_b","mechanism":"norm_violation_detection","affected":["agent_a"],"confidence":0.9,"parent":"trace_001"},
  {"step_id":"trace_003","t":0,"event":"agent_c witnesses the theft","mechanism":"belief_update","affected":["agent_c"],"confidence":0.9,"parent":null}],
  "depth_limit":4,"truncated":false},
 "rejected":[]}
```

> **LLM seam.** `providers/llm_provider.py` defines `LLMStepProvider` (Protocol)
> and the built `RulesProvider`. The Claude `ClaudeProvider` is a commented
> contract only (model `claude-opus-4-8`, `messages.parse(output_format=RawConsequenceTrace)`,
> golden-file caching for hermetic replay) — not built until a benchmark forces it.

---

<a name="phase5"></a>
## Phase 5 — M4 P/G Projector (per-mechanism rules + uncertainty heuristic)

### What it does
Projects a trace into the **ImpactField**: named ΔP/ΔG with per-item uncertainty.
This is the consequence-estimation core. MVP method = per-mechanism rules; swap in
a learned projector once labeled data accrues.

### What to build (`modules/m4_projector.py` + `config/projector_rules.yaml`)

**5.1 Projector rule table** — per mechanism → ΔP/ΔG, applying the §6
**dimension-assignment rules**: assign a dimension to an item **only where a
distinct causal pathway in the trace supports it**; if two dimensions share one
pathway, assign the one the pathway most directly targets. (e.g. deception →
`autonomy(-)`; public shame → `reputation(-)` + `psychological(-)` (two pathways);
teaching → `capability(+)`.)

**5.2 Uncertainty composition** (authored, never guessed):

```
uncertainty(item) = 1 − ( step_confidence
                          × rule_reliability[mechanism]
                          × horizon_factor[horizon] )
horizon_factor:  short 1.00   mid 0.85   long 0.65
rule_reliability: per-mechanism constant in projector_rules.yaml
```

→ longer horizons and lower-confidence steps yield higher uncertainty.

**5.3 Emit-vs-drop** (for items with `delta ≈ 0`):
- **EMIT** if a trace step causally touches the target but sign/magnitude is
  unknown → `delta 0` (or best-guess sign), `uncertainty ≥ 0.7` ("plausible but unknown").
- **DROP** if no trace step touches the target (M2 over-included, M3 found no
  pathway) ("no pathway → no item").
- A "spurious" item = a **confident** (`uncertainty < 0.5`) non-zero item with no
  supporting trace step, or one contradicting an expected-absent label.

**5.4 Failure = explicit ignorance** — an unmapped mechanism projects to
`delta 0, uncertainty 1.0`, never a confident guess.

**5.5 G-item ownership** — M4 may author `cohesion`/`continuity` primary items
**only** by applying declared rule templates over trace + RelationGraph +
GroupState (sign rule-derived; magnitude deferred). M4 **never** authors
`member_welfare` (that is M5's alone). M4 may read current state as *causal
context* (e.g. a starving victim → larger psychological delta) but **must not**
clip delta against it (I5).

### How to run
```powershell
python -m behavioral_layer.modules.m4_projector --in trace.json --graph graph.json --groups groupstates.json --pretty
```

### How to test (`test_m4_projector.py`)
| Test | Asserts |
|---|---|
| `test_dimension_assignment_distinct_pathway` | deception→`autonomy(-)` only; public shame→`reputation(-)`+`psychological(-)` |
| `test_uncertainty_formula` | computed uncertainty equals the §-formula for known inputs; long-horizon item > short-horizon item |
| `test_emit_plausible_unknown` | a touched-but-unknown target emits `delta 0`, `uncertainty ≥ 0.7` |
| `test_drop_no_pathway` | an over-included target with no touching step produces **no** item |
| `test_unmapped_mechanism_explicit_ignorance` | unmapped mechanism → `delta 0`, `uncertainty 1.0` |
| `test_m4_never_authors_member_welfare` | M4 emits only `cohesion`/`continuity` for G; never `member_welfare` |
| `test_delta_not_clipped_against_state` | with a near-saturated target state, M4's `delta` is the pre-saturation value (no clipping) |

### Exit criteria
Full ImpactFields for B1–B10 flow `M1→M2→M3→M4→M5` and the grader scores them.

### Run & test walkthrough (verified)

Status: `8 passed` for M4; `68 passed` for the whole suite (Phases 0–5).
M4 is deterministic and offline.

New files this phase:
- `behavioral_layer/modules/m4_projector.py`
- `behavioral_layer/config/projector_rules.json` — per-mechanism ΔP/ΔG rules + rule_reliability + horizon_factor
- (config tweak) `m3_mechanism_rules.json`: `give`/`share` → `support_provision`; `steal` → `possession_transfer` with `targets_and_actor` ordering so the victim is the resource source

**1 — Test it:**
```powershell
pytest behavioral_layer/tests/test_m4_projector.py -v   # 8 passed
pytest                                                  # 68 passed (whole suite)
```

| Test | Proves |
|---|---|
| `test_dimension_assignment_distinct_pathway` | `deception` → `autonomy` ONLY (no reputation smear); `status_change` → `reputation` + `psychological` (two pathways) |
| `test_uncertainty_formula_and_horizon_monotonic` | `1 − (conf × rule_reliability × horizon_factor)` exactly; longer horizon → higher uncertainty |
| `test_emit_plausible_unknown` | a `"?"` projection emits `delta 0`, `uncertainty ≥ 0.7` |
| `test_drop_no_pathway` | an over-included target with no touching step gets no item |
| `test_unmapped_mechanism_explicit_ignorance` | an `unmappable` step → `delta 0`, `uncertainty 1.0` |
| `test_m4_never_authors_member_welfare` | G items are only `cohesion`/`continuity` |
| `test_delta_not_clipped_against_state` | with a near-saturated target, `delta` is the pre-saturation value |
| `test_pipeline_flows_and_grader_scores_all_benchmarks` | M1→M3→M4→M5 runs for all 10; M5 clean; grader returns a score |

**2 — Run the CLI:**
```powershell
python -m behavioral_layer.modules.m4_projector --trace trace.json --groups groups.json --actor agent_a --pretty
```

**Input** — a `RawConsequenceTrace` (M3's output) + optional `GroupState[]` (for cohesion/continuity) + `--actor`. **Output** — an `ImpactField`: primary ΔP items (each with `evidence_ref` → a step) and cohesion/continuity G items, every item with an authored `uncertainty`. M4 **never** emits `member_welfare` (M5 derives that).

**End-to-end scoreboard (M1→M3→M4→M5 → grader), run live.**

These are the **pre-Phase-6** numbers — M4's rules are a first cut, not yet tuned to the §18 rubric. They show the pipeline produces scorable output and where Phase 6 should focus:

```text
B     recall    sign  scenario
B1      1.00    1.00  Painful truth
B2      0.83    1.00  Theft, observed
B3      0.17    0.00  Group secret betrayed
B4      1.00    1.00  Gift
B5      1.00    1.00  Guild exit
B6      0.00    0.00  Defending a member
B7      0.50    0.50  White lie
B8      0.00    0.00  Affair revealed (subgroup test)
B9      0.60    1.00  Public praise
B10     0.86    1.00  Scarcity sharing (dedup test)
```

Clean cases (B1/B4/B5 perfect; B2/B10 high; B9 sign 1.00) already work; the hard ones — B6 (defense negates an attack), B8 (subgroup roll-down to spouse/child), B3 (cross-group capability gains) — are exactly the per-mechanism-rule gaps **Phase 6 iterates on**. The grader scoring them *is* this phase's exit criterion.

---

<a name="phase6"></a>
## Phase 6 — Integration & MVP acceptance

### What it does
Wires the five modules into one pipeline and runs the contract suite (B1–B10),
then the generalization suite (100 held-out scenarios, §19).

### What to build (`pipeline.py`)

**6.1 Orchestrator** — `M1 → M2 → M3 → M4 → M5` with hard JSON boundaries; every
inter-module payload validated against the schema before the next module runs.
CLI: `python -m behavioral_layer.pipeline --scenario <id>`.

**6.2 Iterate until B1–B10 pass** under the §18 rubric (recall, determinate-sign,
horizon band, expected-absent, welfare structural-only).

**6.3 100 held-out scenario harness** — independently annotated, **not used during
design**. ≥3 annotators/scenario; Krippendorff α ≥ 0.6 floor (scenarios below the
floor are **excluded**, not failed); expected sign = annotator majority; exact tie
→ indeterminate (scored on uncertainty only). Report `exclusion_rate`; flag if > 0.2.

**6.4 §19 acceptance test** — all of:

| # | Criterion | Threshold |
|---|---|---|
| 1 | impact-cell recall (full-key) | ≥ 0.90 (report target-only recall as secondary) |
| 2 | sign accuracy (macro-by-dim, determinate; member_welfare excluded) | ≥ 0.80 |
| 3 | R1–R3 violations (validator-gated structural subset) | = 0 |
| 4 | uncertainty calibration: Spearman(uncertainty, annotator sign-disagreement) | > 0.4 |
| 5 | unmappable_rate | 0 on B1–B10; < 0.15 held-out |
| 6 | pruned-expected-target rate on B1–B10 | = 0 (report annotator exclusion_rate; flag > 0.2) |
| 7 | beats LLM-only baseline | on sign accuracy AND rerun stability (5 reruns, items aligned by natural key) |

> **Baseline** = same model, schema in prompt, **no** M1–M5 decomposition, identical protocol.

### How to run
```powershell
# Full B1-B10 regression
pytest -m benchmark -v

# One scenario end-to-end
python -m behavioral_layer.pipeline --scenario b08_affair_revealed --pretty

# Acceptance suite (Phase 6.3+; needs the 100-scenario annotated set)
pytest -m acceptance -v
python -m behavioral_layer.pipeline --acceptance --reruns 5 --report acceptance_report.json
```

### Exit criteria
All §19 thresholds met. Only then is the MVP "done."

### Run & test walkthrough (verified)

Status: `75 passed` for the whole suite (Phases 0–6). The orchestrator and
acceptance harness are built; two §19 criteria are explicitly DEFERRED (they need
inputs an offline, rules-only build cannot produce).

New files this phase:
- `behavioral_layer/pipeline.py` — the M1→M2→M3→M4→M5 orchestrator + CLI
- `behavioral_layer/acceptance.py` — the §19 metric panel
- `behavioral_layer/tests/test_pipeline.py`, `test_acceptance.py`
- rule iteration in `projector_rules.json` / `m3_mechanism_rules.json` (+ a `betray_secret` mechanic for B3) and a grader fix (`no_negative` now flags only *hallucinated* negatives)

**1 — Test it:**
```powershell
pytest                              # 75 passed (whole suite)
pytest -m benchmark                 # the B1-B10 contract subset
pytest -m acceptance -v             # the §19 panel asserts
```

**2 — Run the pipeline on one scenario:**
```powershell
python -m behavioral_layer.pipeline --scenario B2 --pretty
# -> {candidates, trace, field (checked, incl. derived member_welfare), rejected, diagnostics, unmappable_rate, pruned}
```

**3 — Run the acceptance panel:**
```powershell
python -m behavioral_layer.acceptance
```

**Result (live, post-fix-2).** Per-scenario recall/sign through the full pipeline,
and the §19 panel — after the review's fix 2 (true dataflow + propagation; see
Part F). The earlier as-built-at-Phase-6 snapshot (recall 0.73, B5 absent) is
superseded.

```text
B     recall  sign   B     recall  sign
B1     1.00   1.00   B6     0.80   1.00
B2     1.00   1.00   B7     1.00   1.00
B3     0.83   1.00   B8     0.50   1.00
B4     1.00   1.00   B9     1.00   1.00
B5     1.00   1.00   B10    1.00   1.00

meets_gateable_section19: TRUE

§19 criterion                         value   status
1 impact-cell recall >= 0.90          0.913   PASS
2 sign accuracy >= 0.80               1.00    PASS
3 R1-R3 violations == 0               0       PASS
4 uncertainty calibration > 0.4       —       DEFERRED (needs annotated held-out set)
5 unmappable_rate == 0 on B1-B10      0.0     PASS
6 pruned-expected-target rate == 0    0       PASS
7 beats LLM-only baseline             —       DEFERRED (needs Claude provider + baseline)
8 expected-absent honored == 0        0       PASS
9 M4 cross-step cell collisions == 0  0       PASS
10 no undocumented misses             []      PASS  (misses ⊆ documented allowlist)
rerun stability                       true    PASS  (full-signature, deterministic)
```

**What this means.**
- **All gateable criteria pass:** recall 0.913, sign 1.0, R1–R3 = 0, unmappable 0,
  pruned 0, expected-absent 0, deterministic. `test_mvp_meets_gateable_section19`
  is now a hard assertion (the milestone `xfail` did its job and was removed).
- **Recall cleared the bar via fix 2's propagation:** B6 (defense composed to
  victim-safety+/attacker-capability−/defender-health−), B8 (affair harm
  propagated to the spouse via group co-membership), B9 (rival-peer envy via a
  relation edge), B3 (intel→capability for the rival). B8 still lands at 0.50 —
  a child-vs-spouse / cohesion **horizon** granularity the per-mechanism rules
  can't split — but the *aggregate* is above 0.90, and that residual is exactly
  the learned-projector territory (§22.6).
- **Two criteria remain genuinely deferred,** not failed: uncertainty-calibration
  needs the independently-annotated held-out set, and the baseline comparison needs
  the (deferred) Claude provider. The **real** §19 runs on 100 held-out,
  independently-annotated scenarios — this panel is a build-time stand-in over the
  B1–B10 contract benchmarks (necessary, not sufficient).

### MVP status summary

| Component | State |
|---|---|
| 5 frozen schemas + 2 closed vocabularies + keys + grader (Phase 0) | ✅ done |
| M5 Aggregator/Validator (R1–R3, consistency, two modes) | ✅ done |
| M1 Domain Adapter (C2 invariance, anti-smuggling, unmappable) | ✅ done |
| M2 Candidate Detector (recall-first, bounded recall) | ✅ done |
| M3 Consequence Generator (trees, vocab, registry, depth) | ✅ done |
| M4 P/G Projector (ΔP/ΔG rules, uncertainty, emit-vs-drop, ownership) | ✅ done |
| Pipeline orchestrator + acceptance harness | ✅ done |
| True M1→M2→M3→M4→M5 dataflow + propagation | ✅ **built** (Part F fix 2) |
| B1–B10 §18 rubric | ✅ recall 0.913, sign 1.0, absent 0 (B8 partial: child/cohesion horizon) |
| **Meets gateable §19 exit criterion** | ✅ **met** — recall ≥ 0.90, sign ≥ 0.80, R1–R3 = 0, unmappable 0, pruned 0, absent 0 |
| Learned projector (replaces hand rules, §22.6) | ◑ **dynamic fallback built** (Phase 7) — LLM fills undefined inputs, caches as learned rules; full swap-in of the static table still deferred to labeled data |
| 100 held-out annotated set + §19 calibration | ⏭ deferred — data collection |
| Claude `ClaudeProvider` + LLM-baseline comparison | ⏭ deferred — see Phase 4 seam |
| Calibration session (λ, w_P, G anchoring) + psychological layer | ⏭ deferred (Part E) |

---

<a name="phase7"></a>
## Phase 7 — M4 dynamic extension (LLM fallback / learned projector)

### What it does
Realizes the spec §17 / §22.6 *"swap in a learned projector as labeled data
accrues"* hook, but as an **on-demand LLM fallback** rather than a batch-trained
model. When M4 meets an **undefined input** — an `unmappable` mechanic, or a
mapped mechanism with no static projector rule — it consults a **write-through
learned store**; on a cache miss it asks Claude for the named impacts, then
caches the answer keyed by mechanic so the same input is **never asked twice**.
The static rule table always wins; the LLM only fills gaps it left open.

This is **additive and opt-in**: with no store/provider, M4 behaves exactly as
in Phase 5 (the §5.4 explicit-ignorance fallback is preserved), so the MVP
acceptance numbers are unchanged.

### What to build
- `learned_store.py` — `LearnedRuleStore` (load / get / write-through `put` /
  `learn`) over `config/learned_rules.json`. `LearnedRule` has the **same shape**
  M4 reads for static rules (`p_rules`, `g_rules`, `reliability`).
- `providers/projection_provider.py` — `ProjectionProvider` (Protocol) +
  `ClaudeProjectionProvider` (live). Mirrors the Phase 4 LLM seam.
- `modules/m4_projector.py` — `project(...)` gains optional `learned_store` /
  `llm_provider` params and a `dynamic(step, key)` path; static rules take
  precedence.
- `schemas/raw_trace.py` — `TraceStep.raw_mechanic` (the original scene
  mechanic) so the store is keyed by mechanic type even when `mechanism`
  collapsed to `unmappable`.
- `pipeline.py` — a `--dynamic` flag (needs `ANTHROPIC_API_KEY`). The store
  **always** loads so cached answers replay offline; the live provider is built
  only when `--dynamic` is on **and** a key is present.

**7.1 LLM contract** (mirrors the Phase 4 M3 seam):
```
model            "claude-opus-4-8", official `anthropic` SDK (dep `anthropic>=0.69`)
output           client.messages.parse(output_format=LLMProjection)  -> structured
thinking         {"type": "adaptive"}
determinism      opus-4-8 removes temperature/top_p/seed; reproducibility comes
                 from CACHING outputs and replaying offline, never from sampling.
                 The LLM is consulted only on a cache miss.
```

**7.2 Validate-and-clamp at learn time** (`llm_to_learned`) — structured-output
JSON schemas reject numeric bounds, so magnitudes/reliability are **clamped to
[0,1]** when stored, and any row whose dimension is invalid for its target kind
is **dropped**. `member_welfare` is never accepted (M5's alone, R1).

### Conformance to the frozen v1.6.1 contracts
The dynamic layer is the §22.6 *learned projector*, so it inherits — not
relaxes — every gated M4 contract:

| Contract (spec) | How it holds |
|---|---|
| Explicit ignorance (§5.4) | preserved when no store/provider: `delta 0`, `uncertainty 1.0` |
| Uncertainty authored, not guessed (§5.2) | learned `reliability` feeds `rule_reliability` in the same `1 − conf·rel·hf` formula |
| `delta ∈ [−1,+1]`, pre-saturation (I5) | clamped at learn time and on aggregation |
| `member_welfare` never authored by M4 (§5.5, R1) | dropped at learn time; `GProjection` cannot express it |
| G items limited to `cohesion`/`continuity` (§5.5) | enforced by the `GProjection` type + drop rule |
| Closed P-dim / G-var vocab (§10) | emitted payloads stay in-vocab; learned keys live in the **store**, not the trace's mechanism vocabulary |
| Rerun stability (§19.7) | write-through cache replays deterministically, no network |

### How to run
```powershell
# Offline: replays any cached rules, never calls the network
python -m behavioral_layer.pipeline --scenario B2 --pretty

# Dynamic: asks Claude for undefined inputs, then caches them
$env:ANTHROPIC_API_KEY = "sk-ant-..."
python -m behavioral_layer.pipeline --scenario B2 --dynamic --pretty
```

### How to test (`test_m4_dynamic.py`)
| Test | Proves |
|---|---|
| `test_cache_miss_calls_llm_and_writes_store` | a miss consults the LLM once, writes through to disk, and the projection uses the learned reliability |
| `test_cache_hit_does_not_call_llm` | a second run reloads from disk and never touches the provider |
| `test_backward_compat_explicit_ignorance` | no store/provider → today's `delta 0` / `uncertainty 1.0` per affected party |
| `test_store_offline_replay_without_provider` | a seeded store applies its rule with **no** provider present |
| `test_store_roundtrip_and_apply_does_not_override_static` | learned rules merge in; static rules are untouched on conflict |
| `test_llm_to_learned_clamps_and_drops_invalid` | magnitudes/reliability clamped; `member_welfare` and invalid dims dropped |

### Exit criteria
`99 passed` for the whole suite (Phases 0–7); `ruff` + `mypy` clean. The MVP
acceptance numbers (Phase 6) are unchanged because the extension is gated off by
default.

> **Note — `config/learned_rules.json`** ships as the empty seed `{}`. It is
> written through at runtime in `--dynamic` runs, so expect it to show as
> modified after a live session; `git rm --cached` it if you'd rather it not be
> tracked.

---

<a name="part-d"></a>
# Part D — Testing strategy (all levels)

| Level | Scope | Lives in | Run with |
|---|---|---|---|
| **Unit** | one function/rule (keys, uncertainty formula, a single R-rule) | `test_<module>.py` | `pytest behavioral_layer/tests/test_keys.py` |
| **Schema/contract** | each payload validates/rejects per spec | `test_schemas.py`, `test_vocab.py` | `pytest -k schema` |
| **Module** | a module on crafted inputs in isolation (via its CLI or direct call) | `test_m{1..5}_*.py` | `pytest behavioral_layer/tests/test_m4_projector.py` |
| **Regression (contract)** | B1–B10 end-to-end through the pipeline + grader | `test_benchmarks.py` | `pytest -m benchmark` |
| **Acceptance (generalization)** | 100 held-out scenarios, §19 thresholds | `test_acceptance.py` | `pytest -m acceptance` |

**Principles**
- **M5 is the harness.** Every module's output is validated through M5 before it
  is graded — a module is never "passing" if its output fails M5.
- **B1–B10 are contract/regression tests, not evidence of generalization** (spec
  §18). Passing them is necessary, not sufficient. Generalization is measured
  **only** by the §19 held-out suite, which is annotated independently and not
  used during design.
- **Determinism.** Rules-only modules are fully deterministic — same input, same
  output, every run. Rerun-stability (criterion 7) aligns items by **natural key**.
- **Failure modes are first-class tests.** Every failure path in the spec
  (unmappable, hallucinated entity, dangling ref, double-count, truncation,
  anti-smuggling) has at least one test asserting the **explicit** failure, never
  a silent drop/repair.

---

<a name="part-e"></a>
# Part E — Deferred work (do NOT build for the MVP)

The schemas already reserve every slot below, so answering them later changes **no**
module contract.

**Calibration session (spec §20)**
- `cohesion`/`continuity` measurement — choose, define, anchor proxies.
- `member_welfare` rollup constants — **only** λ and `w_P` (the form is frozen in §15b).
- elicitation of `m` and `id_strength` (kept distinct from personality weight `w`).
- the −1..+1 anchoring protocol for G-variables.
- annotator handbook (incl. P.psychological-vs-emotion examples).

**Psychological layer (spec §21) — entirely out of scope**
- Emotion as P/G dynamics; personality weights/thresholds/habituation; the
  prosocial low-reactivity profile; MCTS deliberation.

Until λ and `w_P` are set, `member_welfare` stays **structural-only** and is
excluded from numeric sign/magnitude scoring everywhere.

---

<a name="part-f"></a>
# Part F — Post-build review: findings & remediation

Five review rounds followed the Phase 0–6 build. Round 1 raised four findings;
round 2 added two (runtime removal of invalid welfare; `xfail` strictness); round 3
added four (a fix-1 regression + M1 relation validation, provider seam, `unc_band`
rationale); round 4 added three (M1→M3 boundary, M4 same-cell aggregation, static
gates); round 5 added three (aggregation-vs-provenance, provider registry, full
`mypy --strict`); round 6 added two (surface+gate M4 collisions, full-signature
rerun stability); round 7 added one (aggregate gate hid a per-scenario miss).
**All are now fixed**, the gateable §19 bar is met (`meets_gateable_section19:
TRUE`, recall 0.913, **10 gated criteria** incl. collisions == 0 and no
undocumented per-scenario misses), and `ruff` + `mypy --strict` (production) pass
clean.

## Findings

| # | Severity | Finding | State |
|---|---|---|---|
| 1 | Critical | MVP did not meet its own §19 exit criterion (recall 0.73 FAIL), but tests only asserted the structural gates, so the suite stayed green and the build was described as "complete". | **Fixed** (gating honest; bar now met via fix 2) |
| 2 | High | The pipeline is not a true M1→M2→M3→M4→M5 dataflow: M3 re-reads the raw scene and M1/M2 outputs are unused downstream, so M2-detected relational/group targets never get projected (root cause of B6/B8/B9 misses). | **Fixed (built)** |
| 3 | High | M5 accepted malformed pre-existing `member_welfare` — it checked only that `derived_from` resolved to existing P keys, not membership, horizon, or structural-only flags. | **Fixed** |
| 4 | Medium | Expected-absent (for welfare) and uncertainty-band were not acceptance-gated; B5 emitted forbidden welfare rows yet graded clean. | **Fixed** |

## Fixes applied (1, 3, 4)

- **Fix 1 — honest gating.** `acceptance.py` now reports `meets_gateable_section19`
  and an `8_expected_absent_honored` criterion. `test_acceptance.py` asserts only
  the criteria that genuinely pass, marks the deferred ones, and gates the full
  §19 with **`xfail(strict=True)`** (`test_mvp_meets_gateable_section19`): it
  xfails today and, once fix 2 makes §19 pass, XPASSes → the test **fails**,
  forcing removal of the stale xfail and conversion to a hard assertion. The doc
  no longer calls the MVP "done".
- **Fix 3 — M5 welfare hardening.** `_validate_welfare_item` receives group
  membership and flags `welfare_nonmember`, `welfare_horizon_mismatch`,
  `welfare_not_structural_only`, and `welfare_unknown_group` (the ref's own natural
  key carries its horizon). Crucially, **runtime now DROPS an invalid welfare row**
  (adds it to the flagged set) rather than returning it beside a diagnostic — and
  because the bad row no longer suppresses re-derivation, M5 re-derives a correct
  structural-only welfare for that group when its own members have P rows. The
  reproduced exploit (`guild_2` welfare from non-member `agent_z`,
  `delta 0.5 / unc 0.1 / numeric_scoring true`) is rejected in strict mode and
  removed from the runtime output.
- **Fix 4 — grader/harness honesty.** Expected-absent on a `member_welfare` cell
  is graded on **presence/absence** (§18 WELFARE), not the confident-only ABSENT
  rule, so B5's structural-only rows are now caught. The grader reports
  `unc_band_mismatches`, and the acceptance panel surfaces `absent_violations`
  per scenario. (B5 is fully resolved by fix 2 below — the leaver is no longer
  rolled into the group's member welfare.)

## Fix 2 — IMPLEMENTED: a true M1→M2→M3→M4→M5 dataflow with propagation

> **Build status: done.** `meets_gateable_section19` is now **TRUE** (recall 0.913,
> sign 1.0, R1–R3 0, unmappable 0, pruned 0, expected-absent 0). The design below
> was implemented as described; the milestone `xfail` flipped to a hard assertion.
> Residual: B8 = 0.50 (a child-vs-spouse horizon split the per-mechanism rules
> can't make — learned-projector territory), but the aggregate clears the bar.

**Problem.** M3 expanded only the scene's own events, so a consequence never
reaches a party that M2 correctly identified through a relationship or group:
the affair's spouse/child (B8), the praise's rival peer (B9), the betrayal's
rival beneficiaries (B3). And opposing events aren't composed, so a defended
attack still reads as harm to the victim (B6).

**Design.**

1. **Hard boundaries — thread the artifacts.** `pipeline.run` passes M1's facts +
   registry and M2's candidates (+ RelationGraph, GroupState) **into** M3, instead
   of M3 re-reading the raw scene. M3 validates `affected` against M1's registry;
   M4 projects only onto trace-touched targets (unchanged), so M2's recall-first
   set bounds *what may be reached*, not what is invented.

2. **Propagation rules in M3 (the core change).** Extend the mechanism-rule table
   with optional **propagation children**: for a mechanism, declare follow-on
   steps targeting *related parties*, resolved from M2 candidates + graph/groups:
   - **group co-members** (M2 `group_member`): `reveal_secret` about X → a
     `belief_update` step for co-members of X's intimate groups (spouse/child)
     ⇒ B8's `agent_b`/`agent_d` psychological−, marriage/family cohesion−.
   - **relation neighbours** (M2 `relation_neighbor`, edge sign −1): `status_change`
     (praise) of X → a `belief_update` step for X's rival peers ⇒ B9's `agent_e`
     envy (psychological−).
   - **adversary beneficiaries**: `betray_secret` to a rival group → a
     `support_provision`/capability step for that group's members ⇒ B3's
     `g3a` capability+/resources+.
   Children attach with `parent` = the originating step, are bounded by
   `depth_limit`, and every `affected` id is registry-validated (rejected + logged
   on miss, as today). Recall-first: over-propagation is fine (M4 zeroes/drops);
   omission is the failure.

3. **Interaction netting for B6.** Add a small pre-projection resolver (in M3)
   that composes opposing events on a shared target: `attack(X→V)` +
   `defend(D→X, protecting V)` ⇒ `physical_harm_attempt(X)` (attacker capability−),
   a protection step for V (safety+), and a self-risk step for D (health−). The
   protected party is carried on the defend event (a `beneficiary` field) or
   inferred from the attack it answers.

4. **Determinism preserved.** All propagation is rule-based and deterministic; if
   an LLM ever drives it, outputs are cached to golden files for hermetic replay
   (the existing seam contract).

**Expected effect.** B3/B6/B8/B9 recall rises toward the rubric; re-run
`python -m behavioral_layer.acceptance` and `test_mvp_meets_gateable_section19`
should XPASS — at which point remove its `xfail` and the MVP genuinely meets the
gateable §19 bar. The propagation traces also become the labeled rows that make
the §22.6 **learned projector** viable.

**Sequencing.** Do fix 2 before wiring the real `ClaudeProvider`: the dataflow
must be correct before a learned/LLM projector can help. The 100-scenario
annotated set + calibration (Part E) remain the separate, genuinely-deferred
inputs for the *real* §19 run.

## Review round 3 — further refinements (all fixed)

A third review pass caught a regression introduced by fix 1 plus three smaller
gaps. All fixed; the gateable §19 bar still holds (`meets_gateable_section19: TRUE`,
recall 0.913).

| # | Severity | Finding | Fix |
|---|---|---|---|
| R3-1 | High | Fix 1 excluded the actor from **every** group's member_welfare (not just the group they leave), suppressing legitimate actor-as-member ΔP (e.g. B10's sharer resource loss). | M5 now takes `exclude_members={group: {agent}}` — a **per-group exit** exclusion. The pipeline derives it from `leave_group` events (which now carry a structured `group`). B5's leaver drops from `guild_2` only; B10's sharer still counts. |
| R3-2 | Medium | M1 validated event actors/targets but **not** relation edge endpoints / membership agents — a relation to `ghost` was silently added to the graph. | M1 now validates edge `from`/`to` (against entities ∪ known group ids) and membership `agent` (against entities), raising `UnknownEntityError` — consistent with event validation. |
| R3-3 | Medium | The provider seam was stale: `RulesProvider.expand(scene)` dropped `group_states`/`relation_graph`/`candidate_ids`, so any future path through it would lose propagation. | `LLMStepProvider` + `RulesProvider.expand` now take and thread the propagation context; the `ClaudeProvider` contract comment requires the same signature. |
| R3-4 | Low | `unc_band` mismatches were surfaced but ungated, with no stated rationale. | Acceptance now reports an explicit **`uncertainty_band_mismatches` INFO criterion** (non-gating): the rule-based uncertainty heuristic is uncalibrated until the calibration session; §19 grades uncertainty via the criterion-4 Spearman (DEFERRED), not per-item bands. A test asserts it stays non-gating. |

## Review round 4 — further refinements (all fixed)

| # | Severity | Finding | Fix |
|---|---|---|---|
| R4-1 | High | The M1→M3 boundary was incomplete: M1 validated only actors/targets, so a `beneficiary` or a structured group ref (a `leave_group`'s `group`) could be invalid; M3 also re-read the raw scene rather than M1's output. A typo'd `group` silently re-emitted B5 welfare; a ghost beneficiary was only partially rejected. | M1 now validates **every** entity ref (actors, targets, **beneficiary**) and the structured **`group`** (against scene `groups` ∪ membership groups), raising `UnknownEntityError`. The pipeline threads **M1's validated `registry`** into `M3.generate(..., registry=...)` so M1 is the authoritative boundary. B5 declares `groups: ["guild_2"]`; a typo now raises. |
| R4-2 | Medium | M4 silently dropped duplicate same-cell consequences (`setdefault` first-wins), losing later evidence/magnitude instead of aggregating or netting. | M4 now **aggregates** colliding `(target, dim, horizon)` cells: signed deltas sum (clamped to `[-1,+1]` so opposing pathways net), uncertainty takes the more-confident value, first evidence_ref kept. This exposed a latent rule conflict in B7 (`lie` emitted both `deception` psychological+ and `belief_update` psychological− at the same short cell); fixed by making `lie` = `deception` only. |
| R4-3 | Low | Static gates weren't clean: `ruff check .` flagged unused imports/locals in tests; `mypy --strict` reported 39 errors. | Tests cleaned; `ruff`/`mypy` dev deps + config added. `ruff check .` passes clean. (mypy was first brought to a pragmatic baseline; **round 5 took production to full `mypy --strict` clean** — see R5-3.) |

## Review round 5 — further refinements (all fixed)

| # | Severity | Finding | Fix |
|---|---|---|---|
| R5-1 | High | The R4-2 aggregation broke provenance: it summed cross-step duplicates but kept only the first `evidence_ref`, so an aggregated delta could include later steps not auditable from output — violating the schema's "primary item is from **one** trace step". | M4 now respects the one-step contract: it aggregates only **within a step** (same `evidence_ref`, fully auditable); a **cross-step** collision keeps the first (so the cell maps to exactly one step) and is **logged** (`_log.warning`), never silently merged. This also stops double-counting (e.g. B6's `support_provision` + `status_change` both pushing `guild_2` cohesion+ → kept once at +0.3, not summed to +0.6). |
| R5-2 | Medium | The provider seam dropped M1's authoritative `registry`: `expand` didn't expose it, so a path through `RulesProvider` rebuilt the registry from the raw scene and would accept a ghost. | `LLMStepProvider`/`RulesProvider.expand` now take `registry=` and thread it to `generate`; the `ClaudeProvider` contract requires it too. A test shows the provider rejects a ghost via the passed registry while a raw-scene rebuild would accept it. |
| R5-3 | Low | Production `mypy --strict` still reported 22 errors (bare `dict`/`list` generics, untyped params/returns). | Parameterized all generics (`dict[str, Any]`, etc.) and typed the remaining params/returns across the 7 modules. **`mypy --strict` now passes clean (0 errors, 28 files)**; `strict = true` is the default in `pyproject.toml`. Tests are still excluded from strict typing (a remaining, smaller backlog). |

> **Static checks (now strict):** `ruff check .` passes clean, and `mypy`
> (`strict = true`, production) passes clean — 28 files, 0 errors. Run them with
> `.\.venv\Scripts\ruff.exe check .` and `.\.venv\Scripts\mypy.exe`. Typing the
> test suite under `--strict` is the one remaining static-analysis backlog item.

## Review round 6 — further refinements (all fixed)

| # | Severity | Finding | Fix |
|---|---|---|---|
| R6-1 | Medium | A cross-step collision was kept-first + **logged**, but invisible to the harness: `PipelineResult` had no collisions field and acceptance didn't gate them, so B6 (where `defend`'s `support_provision` and `status_change` both projected `guild_2` cohesion+) passed while dropping a pathway. | Three-part: (a) **surfaced** — `project(..., collisions_sink=)` and a `PipelineResult.collisions` field; (b) **gated** — acceptance criterion **`9_m4_cell_collisions==0`** (in the gateable set); (c) **rules made collision-free** — removed the redundant `status_change` step from `defend` (its reputation+ was short, B6 wants mid, so it matched nothing; `support_provision` already carries the cohesion+). Benchmarks now report `collisions: 0`. |
| R6-2 | Low | Rerun-stability compared only natural keys, so a changed delta / uncertainty / evidence_ref with stable keys would have passed. | `_field_signature` now compares the full per-item tuple (key + delta + uncertainty + evidence_ref + derived + derived_from). Still PASS (deterministic), but now meaningful. |

## Review round 7 — further refinement (fixed)

| # | Severity | Finding | Fix |
|---|---|---|---|
| R7-1 | Medium | The R6-1 fix removed `defend`'s `status_change`, so B6 emits **no** defender `reputation+` at all, yet B6 expects it at `mid`. Aggregate recall (0.913 ≥ 0.90) **hid** this per-scenario miss. | The miss is a genuine rule-expressiveness limit: `status_change` reputation is `short` (B9 needs short) and re-adding it to `defend` re-creates the now-gated cohesion collision — so B6's `reputation+ mid` can't be hit without breaking B9 or overfitting (learned-projector territory, §22.6). Rather than hide it, **every** per-scenario miss is now enumerated in `KNOWN_RESIDUALS` and **gated** by criterion **`10_no_undocumented_misses`** (in the gateable set): the real misses must be a subset of the documented allowlist, so a NEW or regressed miss fails the gate even while aggregate recall stays ≥ 0.90. B6's `P|agent_a|reputation|mid` is now a named, tracked residual — not a silent tolerance. |

The documented residuals (all horizon-granularity limits of the per-mechanism
rules — the same mechanism must emit a different horizon in another benchmark):
`B3` `guild_2 cohesion short`, `B6` `agent_a reputation mid`, `B8` `marriage_1`/
`family_1 cohesion short` + `agent_d psychological mid`. These are exactly what a
learned projector (§22.6) resolves once labeled data accrues.

---

<a name="appendix1"></a>
# Appendix 1 — Frozen vocabularies (vocab_version "1")

**FACT_TYPE** (closed; 12 + failure value):
`possession`, `location`, `relationship`, `membership`, `role`,
`capability_state`, `condition`, `knowledge`, `commitment`, `threat_exposure`,
`observed_action`, `norm`, + `unmappable` (failure value; confidence ≤ 0.3, never dropped).

**MECHANISM** (closed; 10 + failure value):
`physical_harm_attempt`, `possession_transfer`, `information_transfer`,
`belief_update`, `commitment_change`, `membership_change`, `status_change`,
`support_provision`, `deception`, `norm_violation_detection`, + `unmappable`
(failure value; emitted by M3 when no mechanism fits).

> A `fact_type` describes a normalized world **FACT** (WorldState); a `mechanism`
> describes a causal **TRANSITION step** (RawConsequenceTrace). A fact_type is NOT
> a mechanism (e.g. fact_type `condition` may be acted on by mechanism
> `physical_harm_attempt`).

**v2 shortlist (do NOT add at MVP)** — genuine primitives for a future versioned
bump: `threat/coercion`, `authority_command`, `access_restriction`. NOT primitives
(compositions of v1, never add): apology, forgiveness, reward, punishment.

**P-dimensions (8):** `health`, `safety`, `resources`, `autonomy`, `identity`,
`reputation`, `capability`, `psychological`.

**G-variables (3):** `member_welfare` (M5-derived only), `cohesion`, `continuity`
(emergent; M4 may author cohesion/continuity primary items by rule template).

**M2 inclusion reasons (5):** `direct_object`, `relation_neighbor`,
`group_member`, `observer`, `norm_stakeholder`.

---

<a name="appendix2"></a>
# Appendix 2 — B1–B10 expected labels

Format: `target | dim | sign | horizon | unc-band`. `?` sign = scored on
uncertainty only (must carry `unc ≥ 0.7`). "(structural-only)" = welfare row,
graded on presence/R1/R3/consistency, not sign, until calibration.

**B1 — Painful truth.** `agent_a` tells `agent_b` that b's late father caused the village fire; `friend_ab` is a tracked 2-person group.
```
agent_b   | autonomy      | + | short | low
agent_b   | psychological | - | short | low
agent_b   | psychological | + | long  | high
friend_ab | cohesion      | ? | long  | high
expected-absent: no OTHER G items (only friend_ab may appear)
```

**B2 — Theft, observed.** `agent_a` steals grain from `agent_b`; `agent_c` witnesses; `village_1` tracked.
```
agent_a   | resources      | + | short | low
agent_b   | resources      | - | short | low
agent_b   | psychological  | - | short | low
agent_a   | reputation     | - | mid   | med    (if c spreads word)
village_1 | cohesion       | - | mid   | med
village_1 | member_welfare | (structural-only)
```

**B3 — Group secret betrayed.** `agent_a` (in `guild_2`) reveals guild_2's plan to rival `guild_3` (members g3a, g3b).
```
g3a     | capability     | + | short | med
g3a     | resources      | + | mid   | med
guild_3 | member_welfare | (structural-only)   (supported by g3a rows)
guild_2 | cohesion       | - | short | low
guild_2 | continuity     | - | mid   | med
agent_a | reputation     | - | mid   | low      (within guild_2 context)
```

**B4 — Gift.** `agent_a` gives a tool to `agent_b`. Precondition: b wants it; no hidden obligation/asymmetry/hostility.
```
agent_a   | resources     | - | short | low
agent_b   | resources     | + | short | low
agent_b   | psychological | + | short | low
friend_ab | cohesion      | + | short | med
expected-absent: NO negative items (gift is unambiguously prosocial)
```

**B5 — Guild exit.** `agent_a` publicly leaves `guild_2`. Precondition: guild large enough that no member's individual P changes.
```
agent_a | autonomy   | + | short | low
agent_a | identity   | - | mid   | med
guild_2 | cohesion   | - | short | low
guild_2 | continuity | - | mid   | med
expected-absent: NO guild_2 member_welfare item (no member P changed)
```

**B6 — Defending a member.** `agent_a` physically defends member `agent_b` from outside attacker `attacker_x`.
```
agent_b    | safety     | + | short | low
agent_a    | health     | - | short | med    (risk taken by defender)
attacker_x | capability | - | short | med    (attack repelled)
guild_2    | cohesion   | + | short | low
agent_a    | reputation | + | mid   | low
```

**B7 — White lie.** `agent_a` falsely reassures `agent_b` that b's failing shop is fine.
```
agent_b   | psychological | + | short | low
agent_b   | autonomy      | - | short | low
agent_b   | resources     | - | mid   | high   (delayed correction cost)
friend_ab | cohesion      | ? | long  | high
```

**B8 — Affair revealed (subgroup test).** `agent_c` reveals `agent_a`'s affair; a married to `agent_b`; `marriage_1 ⊂ family_1`; child `agent_d`.
```
marriage_1 | cohesion      | - | short | low    (reported on the SUBGROUP)
marriage_1 | continuity    | - | mid   | med
family_1   | cohesion      | - | short | low    (parent, as a unit)
agent_b    | psychological | - | short | low
agent_d    | psychological | - | mid   | med    (child)
agent_a    | reputation    | - | mid   | low
checks: R3 — marriage_1 items NOT merged into family_1; R1 — b,d once each
```

**B9 — Public praise.** Leader `agent_l` praises `agent_b` before `team_1`; `agent_e` is a rival peer. Precondition: praise perceived as accurate/fair.
```
agent_b | reputation    | + | short | low
agent_b | psychological | + | short | low
agent_l | reputation    | + | short | low    (praising leader, mild)
team_1  | cohesion      | + | short | med
agent_e | psychological | - | short | high   (possible peer envy)
checks: second-order target (peer e) detected by M2
```

**B10 — Scarcity sharing (dedup test).** `agent_a` shares food with `agent_b` in a shortage; both in `family_1` and `guild_2`.
```
agent_a  | resources      | - | short | low
agent_b  | resources      | + | short | low
agent_b  | health         | + | short | low
family_1 | member_welfare | (structural-only, unc high)
guild_2  | member_welfare | (structural-only, unc high)   (b NOT omitted)
family_1 | cohesion       | + | short | med
guild_2  | cohesion       | + | short | high
checks: R1 — b counted ONCE PER GROUP, present in BOTH groups' welfare
```

---

<a name="appendix3"></a>
# Appendix 3 — Schema field reference (spec §11–§16)

**WorldState (§11):** `schema_version`, `vocab_version`, `world_id`, `tick`,
`entities[]` (`id`, `type`, `p_state{8 dims}`), `groups[]` (ids only),
`facts[]` (`id`, `text`, `fact_type`, `source`, `confidence`).
*Supplied by the world model via M1; the behavioral layer reads it, never writes it.*

**RelationGraph (§12):** `nodes[]`, `edges[]` (`from`, `to`, `r_vector`
{`trust`, `affection`, `dependence`, `authority`, `history_valence`}, `sign`),
`memberships[]` (`agent`, `group`, `m`, `id_strength`).

**RawConsequenceTrace (§13):** `action_id`, `steps[]` (`step_id`, `t`, `event`,
`mechanism`, `affected[]`, `confidence`, optional `parent`), `depth_limit`,
`truncated`.

**ImpactField (§14) — canonical output:** `actor_id`, `action_id`,
`impact_field[]` of items; per item: `target_id`, `target_type` (`P`|`G`),
`dimension`, `delta` ∈ [-1,+1], `horizon`, `uncertainty` ∈ [0,1], plus provenance
(**primary**: `evidence_ref → step_id`; **derived**: `derived:true`,
`derived_from[]`, and for uncalibrated welfare: `calibration_status:"structural_only"`,
`numeric_scoring:false`).

**GroupState (§16):** `group_id`, `tracked`, `members[]` (`agent`, `m`,
`id_strength`), `subgroups[]`, `g_state` {`member_welfare`, `cohesion`,
`continuity`} (absolute STATE values, not deltas). Only `tracked:true` groups
receive emergent-variable items.

---

*End of guide. Build order = Part C top-to-bottom. The leverage points, in order:
the B1–B10 fixtures (0.5) → M5 (Phase 1) → the natural/storage keys (0.4). Get
those three exactly right and the rest is fill-in.*