# P/G Behavioral Layer — Run & Test Runbook

> Operator guide for the built MVP. Every command below is real and was run on
> this repo. Companion to `MindOrigin_Behavioral_Layer_Implementation_Plan.md`
> (the design/build guide); this file is purely *how to run and test it*.
>
> **Current verified state:** `93 passed` · `ruff` clean · `mypy --strict`
> (production) clean · acceptance `meets_gateable_section19: True`.
>
> **Platform:** Windows + PowerShell, repo at `C:\mindorigin`, venv at `.venv`.
> Commands use `.\.venv\Scripts\python.exe …` so they work without activating.
> On macOS/Linux substitute `.venv/bin/python`.

---

## Table of contents

- [0. The 60-second smoke test](#0)
- [1. Prerequisites](#1)
- [2. One-time setup](#2)
- [3. Running the test suite](#3)
- [4. Tests by level (unit → acceptance)](#4)
- [5. Static gates (ruff, mypy)](#5)
- [6. Run the whole pipeline (one command)](#6)
- [7. Run each module's CLI in isolation](#7)
  - [M1 Domain Adapter](#m1)
  - [M2 Candidate Target Detector](#m2)
  - [M3 Raw Consequence Generator](#m3)
  - [M4 P/G Projector](#m4)
  - [M5 Aggregator / Validator](#m5)
- [8. The acceptance harness (§19 panel)](#8)
- [9. Export the JSON Schemas](#9)
- [10. Reading the outputs](#10)
- [11. Troubleshooting](#11)
- [12. One-page cheat sheet](#12)

---

<a name="0"></a>
## 0. The 60-second smoke test

From `C:\mindorigin`, after [setup](#2):

```powershell
.\.venv\Scripts\python.exe -m pytest -q
.\.venv\Scripts\python.exe -m behavioral_layer.acceptance
```

Expect `93 passed`, and an acceptance report ending with `"meets_gateable_section19": true`.
If both hold, the system is healthy.

---

<a name="1"></a>
## 1. Prerequisites

| Need | Check |
|---|---|
| Python 3.11+ | `python --version` (repo built on 3.13) |
| The repo | `C:\mindorigin\behavioral_layer\` exists |
| venv | `C:\mindorigin\.venv\` exists (else see setup) |

No network and **no LLM key** are required — the MVP is rules-only and fully offline.

---

<a name="2"></a>
## 2. One-time setup

Only needed if `.venv` doesn't exist yet. From `C:\mindorigin`:

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install -e ".[dev]"
```

The `[dev]` extra installs `pytest`, `pytest-cov`, `ruff`, and `mypy`. Verify:

```powershell
.\.venv\Scripts\python.exe -c "import behavioral_layer, pydantic, pytest; print('ok')"
```

**Optional — activate the venv** so you can drop the `.\.venv\Scripts\` prefix:

```powershell
.\.venv\Scripts\Activate.ps1     # then: python -m pytest -q, ruff check ., mypy
```

(If activation is blocked: `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned`.)

---

<a name="3"></a>
## 3. Running the test suite

```powershell
# Everything (93 tests)
.\.venv\Scripts\python.exe -m pytest

# Quiet one-line-per-file summary
.\.venv\Scripts\python.exe -m pytest -q

# Verbose: every test name + PASS/FAIL
.\.venv\Scripts\python.exe -m pytest -v

# With coverage
.\.venv\Scripts\python.exe -m pytest --cov=behavioral_layer --cov-report=term-missing
```

Useful flags: `-x` (stop at first failure), `-k <substr>` (run matching tests,
e.g. `-k welfare`), `--lf` (last-failed), `behavioral_layer/tests/test_m5_validator.py::test_r1_member_counted_once_per_group` (one test).

**Expected tail:** `93 passed in ~0.6s`.

---

<a name="4"></a>
## 4. Tests by level

The 11 test files, what each proves, and how to run just that file:

| File | Layer | Proves | Run just it |
|---|---|---|---|
| `test_schemas.py` | contract | the 5 frozen schemas round-trip spec examples; reject out-of-range / unknown-field / bad-provenance | `pytest behavioral_layer/tests/test_schemas.py` |
| `test_vocab.py` | contract | the 2 closed vocabularies + dim enums are complete; junk tokens raise | `…/test_vocab.py` |
| `test_keys.py` | unit | `natural_key`/`storage_key` exact format; no P/G collision | `…/test_keys.py` |
| `test_grading.py` | unit | the §18 grader (recall, sign, ABSENT, EXTRA) | `…/test_grading.py` |
| `test_m5_validator.py` | module | M5 R1–R3, welfare↔member-P, structural-only, two modes, invalid-welfare removal | `…/test_m5_validator.py` |
| `test_m1_adapter.py` | module | C2 invariance, unmappable, anti-smuggling, relation/beneficiary registry | `…/test_m1_adapter.py` |
| `test_m2_targets.py` | module | recall-first, antagonist/second-order, bounded-recall pruning logged | `…/test_m2_targets.py` |
| `test_m3_consequence.py` | module | trees, vocab, registry, depth, propagation, provider seam | `…/test_m3_consequence.py` |
| `test_m4_projector.py` | module | ΔP/ΔG rules, uncertainty formula, emit-vs-drop, collision provenance; **end-to-end B1–B10 → grader** | `…/test_m4_projector.py` |
| `test_pipeline.py` | integration | full M1→M5 orchestration, determinism, propagation reaches relational/group parties | `…/test_pipeline.py` |
| `test_acceptance.py` | acceptance | the §19 panel — structural gates, deferred criteria, no undocumented misses | `pytest -m acceptance` |

**Markers.** Only `acceptance` is used. Run the §19 panel asserts with:
```powershell
.\.venv\Scripts\python.exe -m pytest -m acceptance -v
```
(The `benchmark` marker is declared in `pyproject.toml` but currently unused — the
B1–B10 contract coverage lives in `test_m4_projector.py` (end-to-end) and
`test_acceptance.py`.)

---

<a name="5"></a>
## 5. Static gates (ruff, mypy)

Both pass clean and are part of the quality bar.

```powershell
# Lint (import order, unused, pyflakes). Auto-fix with --fix.
.\.venv\Scripts\ruff.exe check .
.\.venv\Scripts\ruff.exe check --fix .

# Types — production is strict (config: strict = true; tests excluded)
.\.venv\Scripts\mypy.exe
```

**Expected:** `All checks passed!` and `Success: no issues found in 28 source files`.
Config lives in `pyproject.toml` (`[tool.ruff]`, `[tool.mypy]`).

---

<a name="6"></a>
## 6. Run the whole pipeline (one command)

The orchestrator runs **M1 → M2 → M3 → M4 → M5** on a built-in benchmark scenario
(`B1`…`B10`) loaded from the test fixtures:

```powershell
.\.venv\Scripts\python.exe -m behavioral_layer.pipeline --scenario B2 --pretty
```

| Flag | Meaning |
|---|---|
| `--scenario` | benchmark id `B1`…`B10` (case-insensitive) |
| `--mode` | `runtime` (default; returns valid items + diagnostics) or `strict` (raises on any violation) |
| `--pretty` | indent the JSON |

**Output** — a `PipelineResult` JSON with:
- `candidates` — M2's affected entities/groups (`target_id`, `inclusion_reason`, `is_group`)
- `trace` — M3's `RawConsequenceTrace` (the causal tree)
- `field` — the **checked ImpactField** (M4's ΔP/ΔG + M5-derived `member_welfare`)
- `rejected` — M3 steps dropped (e.g. unregistered entity), with reasons
- `diagnostics` — M5 diagnostics (empty when clean)
- `collisions` — M4 cross-step cell collisions (empty for the benchmarks)
- `unmappable_rate` — M1 fraction of `unmappable` facts (0 on benchmarks)
- `pruned` — M2 candidates pruned by caps (empty under recall-first defaults)

Try a few: `--scenario B6` (defense), `--scenario B8` (affair → spouse/child
propagation), `--scenario B10` (sharing → per-group welfare).

---

<a name="7"></a>
## 7. Run each module's CLI in isolation

Every module exposes a `__main__` so you can drive one stage on a JSON file. Each
reads JSON from disk and prints JSON to stdout. **Create the JSON with a real
editor or the `Write` tool — not `Set-Content -Encoding utf8` (it adds a UTF-8 BOM
that breaks JSON parsing; see [Troubleshooting](#11)).**

<a name="m1"></a>
### M1 Domain Adapter — scene → facts + RelationGraph + registry

```powershell
.\.venv\Scripts\python.exe -m behavioral_layer.modules.m1_adapter --scene scene.json --pretty
# optional: --world path\to\mapping.json   (default: config/worlds/npc_village_01.json)
```
`scene.json`:
```json
{"world_id":"npc_village_01","tick":1,
 "entities":[{"id":"agent_a","type":"person"},{"id":"agent_b","type":"person"}],
 "events":[{"id":"f1","text":"agent_a steals grain from agent_b","raw_mechanic":"steal","actors":["agent_a"],"targets":["agent_b"]}]}
```
Output: `{ "facts": [...], "relation_graph": {...}, "registry": [...], "unmappable_rate": 0.0 }`.
Try the C2 demo: swap `raw_mechanic` to `firebolt` and the `text` — the `fact_type`
stays `condition`, only the free text changes.

<a name="m2"></a>
### M2 Candidate Target Detector — action (+ graph + groups) → candidates

```powershell
.\.venv\Scripts\python.exe -m behavioral_layer.modules.m2_targets --action action.json --graph graph.json --pretty
# --graph and --groups are optional
```
`action.json`: `{"actor":"agent_l","direct_objects":["agent_b"]}`
`graph.json`:
```json
{"nodes":["agent_l","agent_b","agent_e"],
 "edges":[{"from":"agent_b","to":"agent_e","r_vector":{"trust":0.3,"affection":0.0,"dependence":0.0,"authority":0.0,"history_valence":-0.4},"sign":-1}]}
```
Output: `candidates` (each with `inclusion_reason`) + `pruned`. Here `agent_e` is
surfaced as a `relation_neighbor` (second-order target).

<a name="m3"></a>
### M3 Raw Consequence Generator — scene → RawConsequenceTrace

```powershell
.\.venv\Scripts\python.exe -m behavioral_layer.modules.m3_consequence --scene scene.json --pretty
# optional: --rules path\to\m3_mechanism_rules.json
```
Reuse the M1 `scene.json` (theft). Output: `{ "trace": {steps:[...]}, "rejected": [] }`
— a **tree**: `possession_transfer` (root) → `norm_violation_detection` (child).
(Standalone M3 has no group/graph context, so cross-event *propagation* only runs
inside the full pipeline.)

<a name="m4"></a>
### M4 P/G Projector — trace (+ groups) → ImpactField

```powershell
.\.venv\Scripts\python.exe -m behavioral_layer.modules.m4_projector --trace trace.json --groups groups.json --actor agent_a --pretty
```
`trace.json` is an M3 output (a `RawConsequenceTrace`). `groups.json` is a
`GroupState[]` (see M5 below). Output: an `ImpactField` of ΔP/ΔG items, each with
an authored `uncertainty` and an `evidence_ref`. M4 never emits `member_welfare`.

<a name="m5"></a>
### M5 Aggregator / Validator — ImpactField (+ groups) → checked field

```powershell
.\.venv\Scripts\python.exe -m behavioral_layer.modules.m5_validator --in field.json --groups groups.json --mode strict --pretty
# optional: --trace trace.json   (enables dangling evidence_ref checking)
```
`field.json` (M4 output: primary items, **no** member_welfare):
```json
{"actor_id":"agent_a","action_id":"act_017","impact_field":[
  {"target_id":"agent_a","target_type":"P","dimension":"resources","delta":-0.3,"horizon":"short","uncertainty":0.2,"evidence_ref":"t1"},
  {"target_id":"agent_b","target_type":"P","dimension":"resources","delta":0.3,"horizon":"short","uncertainty":0.2,"evidence_ref":"t1"}]}
```
`groups.json`:
```json
[{"group_id":"village_1","tracked":true,
  "members":[{"agent":"agent_a","m":0.9,"id_strength":0.8},{"agent":"agent_b","m":0.9,"id_strength":0.7}],
  "g_state":{"member_welfare":0.0,"cohesion":0.0,"continuity":0.0}}]
```
Output: `{ "ok": true, "items": [...incl. derived village_1 member_welfare...], "diagnostics": [] }`.
- `--mode strict` exits non-zero and prints `REJECTED (strict):` on any violation.
- `--mode runtime` returns the valid items + a `diagnostics` list (drops only the bad ones).

---

<a name="8"></a>
## 8. The acceptance harness (§19 panel)

Runs the full pipeline over all 10 benchmarks and computes the §19 metric panel:

```powershell
.\.venv\Scripts\python.exe -m behavioral_layer.acceptance
```

Top-level `meets_gateable_section19` is the headline. The `criteria` block:

| Criterion | Gateable? | Meaning |
|---|---|---|
| `1_impact_cell_recall>=0.90` | ✅ | full-key recall over expected cells (≈ 0.913) |
| `2_sign_accuracy>=0.80` | ✅ | macro-by-dimension sign on determinate items (1.0) |
| `3_r1_r3_violations==0` | ✅ | M5 R1–R3 / consistency violations |
| `4_uncertainty_calibration>0.4` | ⏸ DEFERRED | needs the annotated held-out set |
| `5_unmappable_rate==0_on_b1_b10` | ✅ | M1 produced no `unmappable` |
| `6_pruned_expected_target_rate==0` | ✅ | M2 never pruned an expected target |
| `7_beats_llm_baseline` | ⏸ DEFERRED | needs the Claude provider + baseline |
| `8_expected_absent_honored==0` | ✅ | no forbidden cell emitted (incl. structural-only welfare) |
| `9_m4_cell_collisions==0` | ✅ | no silent cross-step cell drop |
| `10_no_undocumented_misses` | ✅ | per-scenario misses ⊆ the documented `KNOWN_RESIDUALS` allowlist |
| `uncertainty_band_mismatches` | INFO | reported, non-gating (heuristic uncalibrated) |
| `rerun_stability` | ✅ | identical full item-signatures across 2 runs |

`per_scenario` shows each benchmark's `recall`/`sign`/`missing`/`collisions`.
Criteria **4** and **7** are DEFERRED by design — the *real* §19 runs on 100
held-out, independently-annotated scenarios with an LLM baseline; this panel is a
build-time stand-in over the B1–B10 contract benchmarks.

Run the panel's assertions as tests: `.\.venv\Scripts\python.exe -m pytest -m acceptance -v`.

---

<a name="9"></a>
## 9. Export the JSON Schemas

```powershell
.\.venv\Scripts\python.exe -m behavioral_layer.schemas --export-json-schema schemas_out\
```
Writes `WorldState.json`, `RelationGraph.json`, `RawConsequenceTrace.json`,
`ImpactField.json`, `GroupState.json`. (`schemas_out\` is a regenerable artifact;
safe to delete.)

---

<a name="10"></a>
## 10. Reading the outputs

**ImpactField item** (the canonical output cell):
- `target_id` / `target_type` (`P`|`G`) / `dimension` / `horizon` — the natural key.
- `delta` ∈ [-1,1] — projected, **pre-saturation** change (not clipped to current state).
- `uncertainty` ∈ [0,1] — `1 − (step_confidence × rule_reliability × horizon_factor)`.
- `evidence_ref` — a trace `step_id` (primary items); or `derived:true` + `derived_from` (member_welfare).
- `calibration_status:"structural_only"` + `numeric_scoring:false` on uncalibrated welfare (delta 0, unc 1.0).

**M5 diagnostic codes** (when not clean): `m4_authored_member_welfare`,
`dangling_evidence_ref`, `emergent_on_untracked_group`, `welfare_without_member_p`,
`welfare_double_count`, `welfare_parent_summing`, `welfare_nonmember`,
`welfare_horizon_mismatch`, `welfare_not_structural_only`, `welfare_unknown_group`.

**M3 `rejected` reasons:** `unregistered:[…]` (hallucinated entity), `parent_missing`.

---

<a name="11"></a>
## 11. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `Invalid JSON: expected value … ﻿` | You wrote the JSON with `Set-Content -Encoding utf8` (UTF-8 **BOM**). Use a normal editor, or `[System.IO.File]::WriteAllText("$PWD\f.json", $json)`. |
| `python -c "..."` SyntaxError on Windows | PowerShell mangles inner quotes. Put the code in a `.py` file and run that, or use a single-quoted here-string `@' … '@`. |
| `pytest` "no tests ran" / exit 5 | You filtered with `-k`/`-m` that matched nothing (e.g. `-m benchmark` — unused marker). |
| `ruff`/`mypy` "not recognized" | Use the venv path: `.\.venv\Scripts\ruff.exe` / `.\.venv\Scripts\mypy.exe`, or activate the venv. |
| A `§`/`→` shows as `?`/`�` in console output | Console codepage (cp1252), display-only — the data is fine. `chcp 65001` for UTF-8 if needed. |
| Activation script blocked | `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned`. |
| `mypy` flags a test file | Production is strict; tests are excluded from strict typing by design (`exclude` in `[tool.mypy]`). |

---

<a name="12"></a>
## 12. One-page cheat sheet

```powershell
# --- from C:\mindorigin ---
.\.venv\Scripts\Activate.ps1                          # (optional) then drop the prefix

# tests
.\.venv\Scripts\python.exe -m pytest -q               # all 93
.\.venv\Scripts\python.exe -m pytest -v               # verbose
.\.venv\Scripts\python.exe -m pytest -m acceptance -v # §19 panel asserts
.\.venv\Scripts\python.exe -m pytest --cov=behavioral_layer --cov-report=term-missing

# static gates
.\.venv\Scripts\ruff.exe check .
.\.venv\Scripts\mypy.exe

# run the system
.\.venv\Scripts\python.exe -m behavioral_layer.pipeline --scenario B2 --pretty
.\.venv\Scripts\python.exe -m behavioral_layer.acceptance

# one module at a time
.\.venv\Scripts\python.exe -m behavioral_layer.modules.m1_adapter   --scene scene.json --pretty
.\.venv\Scripts\python.exe -m behavioral_layer.modules.m2_targets   --action action.json --graph graph.json --pretty
.\.venv\Scripts\python.exe -m behavioral_layer.modules.m3_consequence --scene scene.json --pretty
.\.venv\Scripts\python.exe -m behavioral_layer.modules.m4_projector --trace trace.json --groups groups.json --actor agent_a --pretty
.\.venv\Scripts\python.exe -m behavioral_layer.modules.m5_validator --in field.json --groups groups.json --mode strict --pretty

# schemas
.\.venv\Scripts\python.exe -m behavioral_layer.schemas --export-json-schema schemas_out\
```

**Healthy =** `93 passed` · `All checks passed!` · `Success: no issues found` ·
`"meets_gateable_section19": true`.
