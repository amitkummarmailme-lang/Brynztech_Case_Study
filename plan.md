# Applied AI Case Study: End-to-End Solution Plan

Source brief: [Applied_AI_Case_Study.md](Applied_AI_Case_Study.md). This plan covers every requirement in it.
Working name for the package: `roomscan`.

---

## 0. Open decisions (confirm before Phase 0 ends)

| # | Decision | Recommended default | Why |
|---|----------|---------------------|-----|
| D1 | Capture route | **Route 2: stock capture protocol** | Dev machine is Windows. Route 1 needs a Mac, Xcode and an Apple developer account for TestFlight. Route 2 puts the effort into the pipeline, which carries 55%+ of the score. |
| D2 | Stock LiDAR capture app | Decide in the Phase 0 spike: *3d Scanner App*, *Record3D*, or Polycam raw export | The app must export per-frame RGB, depth, pose and intrinsics. Verify this on a real device, don't assume it. |
| D3 | Compute for inference and the walk-in test | Assume a CPU laptop plus an optional GPU. Every model needs a CPU-feasible path and a time budget. | The walk-in runs "cold, in front of us", probably on our hardware. |
| D4 | iPhone access | Need a Pro-class phone (LiDAR) and ideally a non-Pro iPhone 15 | The device matrix must say which tier runs on which hardware, and the benchmark must back that up. |
| D5 | Deadline | Unknown | The roadmap in section 14 is in phases, not dates. Add dates once known. |
| D6 | Head-to-head app | magicplan or Polycam (free tier) | Pick one in Phase 0 and record the exact version. |

---

## 1. Scoring-driven priorities

| Weight | Component | What it means for the build |
|-------:|-----------|------------------------------|
| 30% | Walk-in test | Robustness on unseen data comes first. The pipeline must have no benchmark-specific tuning and a `doctor` pre-flight. It also needs a rehearsed runbook, graceful degradation, and fast, deterministic behaviour. |
| 25% | Fix loop | Run a baseline early, pre-register the fix, ship it, and keep a regenerable before and after. This needs a sound, quick-to-iterate pipeline and honest diagnosis. |
| 15% | Benchmark accuracy, all 3 tiers | Collect real benchmark data early and run the gates continuously in CI. |
| 10% | Compliance matrix | Maintain it from Phase 1, with an automated path-existence check. |
| 10% | Head-to-head vs incumbent app | Capture the same two rooms with the incumbent during benchmark capture, not afterwards. |
| 5% | Capture route quality | Write the protocol, then user-test it on someone who is not an engineer. |
| 5% | Process evidence | Commit continuously, with a real history. See section 13. |

Hard caps and zeros:
- Confident garbage on thin input caps the total score, so calibration and quality-aware interval inflation are first-class features.
- "Poses used as-is" on the drift row is an automatic fail.
- A photo path that handles single rooms only fails its row.
- Analysis with no shipped fix scores zero on the fix loop.
- A fix with no regenerable before and after scores zero.
- A repo materialising in 1-2 commits scores zero on process evidence.

---

## 2. Requirement traceability (summary)

The full matrix lives in `docs/compliance_matrix.md` and is checked by `tools/check_compliance.py`. This is the skeleton.

| Brief requirement | Planned artifact |
|---|---|
| Part 1: capture route, 3 tiers, device matrix | `capture/PROTOCOL.md` (one page), `docs/device_matrix.md`, `src/roomscan/ingest/*` |
| Part 2: output contract (rooms, stitch, damage, concealed, scope, CIs, JSON schema, render, one command) | `schema/capture.schema.json`, `src/roomscan/output/*`, `roomscan scan <dir>` |
| Part 2: benchmark composition | `bench/data/*`, `bench/MANIFEST.md`, `bench/ground_truth/*` |
| Metric, repeatability, drift and photo-stitch gates | `bench/gates.py`, `bench/repeatability.py`, `bench/ablation_drift.py` |
| Calibration scored at every tier | `src/roomscan/uncertainty/*`, `docs/calibration.md` |
| Part 3: head-to-head | `bench/headtohead/`, `docs/headtohead.md` |
| Part 4: fix loop | `docs/fix_declaration.md`, tags `before-fix` and `after-fix`, `make fixloop` |
| Part 5: process evidence | git history, `docs/decisions/` |
| Deliverables 1-8 | See section 12 |
| Walk-in test | `roomscan doctor`, `docs/walkin_runbook.md` |
| Constraints: disclosure, offline, large weights by script | `docs/DISCLOSURES.md`, `scripts/fetch_models.py` |
| Mirrors, glass, wet-look, low light | `src/roomscan/ingest/quality.py`, `geometry/hazards.py`, `docs/failure_modes.md` |

---

## 3. Architecture

### 3.1 Stack
- Python 3.11 with `uv`, which gives a lockfile and fast clean-machine setup. `pydantic` v2 models are the single source of truth for the output schema, and the published JSON Schema is generated from them.
- Geometry and optimisation: `numpy`, `scipy`, `open3d`, `shapely`, and `gtsam` or `scipy.optimize` for pose graphs (decide in Phase 6; use a pure-scipy fallback if `gtsam` wheels are a pain on Windows).
- Vision models (all pretrained, all disclosed, all run locally):
  - Multi-view geometry: VGGT, MASt3R or DUSt3R, or COLMAP/GLOMAP.
  - Monocular metric depth: Depth Pro, UniDepth or MoGe.
  - Segmentation: SAM 2.
  - Open-vocabulary detection: Grounding DINO, OWLv2 or YOLO-World.
  - Damage classification: CLIP or a local VLM.
  - Final choices come from a bake-off in the relevant phase, recorded as an ADR.
- Rendering: matplotlib to SVG and PNG, with an HTML viewer as stretch.
- Tooling: `pytest`, `ruff`, `mypy`, `Makefile`, GitHub Actions for lint, unit tests and the gate smoke test.

### 3.2 Repo layout
```
.claude/                    project memory and must-do checklist
capture/PROTOCOL.md         one-page non-engineer protocol
schema/                     published JSON Schema + examples
src/roomscan/
  cli.py                    one command per capture: roomscan scan <dir>
  pipeline.py               orchestrator (DAG of stages, caching, timing)
  ingest/                   tier_detect, photos, video, lidar adapters, quality gates
  recon/                    gravity, sfm/poses, metric_depth, scale fusion, depth fusion
  geometry/                 planes, room_layout, ceiling, openings, adjacency, hazards
  stitch/                   door_match, pose_graph, loop_closure, plane_anchor, overlap_check
  damage/                   proposals, classify, extent (metric), concealed_rules + rules/*.yaml
  scope/                    catalog.yaml, mapper
  uncertainty/              error_budget, conformal, calibrate, quality_inflation
  output/                   schema (pydantic), export_json, render_plan
  models/                   registry, fetch (weights by script), pinned hashes
  cache/                    deterministic replay of model outputs
bench/                      data, ground_truth, gates, repeatability, ablations, headtohead, run_benchmark
docs/                       compliance_matrix, device_matrix, decisions (ADRs), calibration, failure_modes,
                            fix_declaration, DISCLOSURES, walkin_runbook, technical_report
scripts/                    fetch_models, make_repro_bundle, doctor
tests/                      unit, integration, golden
tools/                      check_compliance
```

### 3.3 Pipeline (one command: `roomscan scan <capture_dir> --out <dir> [--tier auto|photo|video|lidar] [--live|--replay]`)

```
ingest -> tier detect -> quality gates -> [tier recon] -> gravity-aligned metric geometry
-> per-room layout (walls, floor, ceiling, openings) -> damage -> concealed rules -> scope
-> multi-room stitch + drift correction -> uncertainty + calibration -> JSON + render
```

Principle: all three tiers converge on one internal representation, a **gravity-aligned metric scene per room** (a point set or depth, plus camera poses, plus a per-quantity uncertainty). Everything downstream is tier-agnostic. This is what makes "same output contract from each tier" cheap, and the intervals widen automatically because upstream uncertainty is larger.

---

## 4. Tier design

### 4.1 LiDAR tier (Pro-class; strict gates)
- Ingest per-frame RGB, depth, confidence, pose and intrinsics from the stock app export.
- Pipeline: confidence-filter depth, fuse with TSDF or a point cloud, and align gravity from poses.
- Fit planes with RANSAC plus refinement. Classify them as floor, ceiling and walls. Intersect the walls to build the room polygon, then read ceiling height as the floor-to-ceiling plane distance.
- Targets: ceiling height ≤ 1.5 cm, opening width ≤ 2 cm on ≥ 85% of openings.
- Error budget: LiDAR depth noise, pose drift, and plane-fit residual (propagated).

### 4.2 Video tier (±3% wall lengths)
- Extract keyframes by sharpness and parallax selection. Estimate poses with a multi-view model or SfM using iPhone intrinsics from metadata.
- Metric scale comes from fusing several cues, each with its own uncertainty: monocular metric depth, standard door height, and camera height priors. Fusion is inverse-variance weighted.
- Then follows the LiDAR geometry path with a noisier input.

### 4.3 Photo tier (±8%; the floor, any picture in, results out)
- Input is a folder per room with 2-8 stills.
- Poses come from a feed-forward multi-view model (small-baseline robust). Intrinsics come from EXIF.
- Scale comes from the same multi-cue fusion as video, with much larger uncertainty.
- **Whole-property stitch from photos** (a hard requirement). Each room folder must include a *doorway shot* per connector (specified in the protocol). Adjacency is inferred by matching door and opening geometry and appearance across rooms, with shared-opening width as a metric consistency check. A tiny optional `adjacency.yaml` hint is accepted, but the pipeline must produce a plan without it, and the report states which was used.
- Rooms are placed by SE(2) pose graph optimisation. A non-overlap check rejects or repairs infeasible layouts.

### 4.4 Device matrix (deliverable)
`docs/device_matrix.md` is filled from benchmark evidence, not from assumptions. It lists tier × device × measured error × interval coverage. LiDAR needs Pro-class devices. Photo and video run on any iPhone 15 or newer.

---

## 5. Output contract

Per capture, JSON validated against `schema/capture.schema.json`:
- `property`: stitched plan (rooms, transforms, adjacency graph, footprint, area) with intervals.
- `rooms[]`: walls (length, thickness if known), ceiling height, floor area, `openings[]` (type, width, height, position, wall), each measurement as `{value, lo, hi, confidence_level, method}`.
- `surfaces[]`: stable IDs for walls, floor and ceiling.
- `damage[]`: surface ID, class, mask or polygon in surface coordinates, metric extent with interval.
- `concealed_flags[]`: surface or region, **rule ID and the evidence that fired it**.
- `scope_items[]`: keyed to surface IDs, with quantity, unit and catalog code.
- `provenance`: tier, device, model versions and hashes, quality flags, timing.
- A rendered dimensioned plan (SVG and PNG) with intervals shown.

Schema conformance is enforced in tests, and a golden-output test checks deterministic replay.

---

## 6. Uncertainty and calibration (scored at every tier)

1. **Error budget** per quantity: propagate depth noise, pose uncertainty, scale uncertainty and fit residuals to a standard deviation (`docs/error_budget.md`).
2. **Quality-aware inflation**: blur, low-light, texture-poor, mirror and glass indicators, low parallax, and few photos each widen the intervals via a learned or tabulated factor. Thin input never yields narrow intervals.
3. **Conformal calibration**: split-conformal or CV+ on benchmark residuals, per tier, so the nominal 90% interval is empirically about 90%. The calibrator is frozen before the walk-in. There is no tuning on walk-in data.
4. **Calibration report**: coverage vs nominal, interval width vs error, and a reliability plot per tier. It also states bias vs spread explicitly, because the brief requires saying which problem a tier has (repeatable-but-biased or unrepeatable).
5. **Refusal and flag path**: if quality gates fail hard, output wide intervals plus a `low_confidence` flag. The pipeline never crashes or returns narrow garbage.

---

## 7. Drift accountability

- Multi-room LiDAR and video: pose graph with (a) loop closure when revisiting the connector or start room, (b) plane-anchored correction (walls are vertical, floors are level, and walls in a room are mostly orthogonal or parallel, applied as soft priors), and (c) shared-opening constraints (the doorway seen from both rooms must have equal width and coincident position).
- Photo tier: room-level SE(2) pose graph with door-match constraints, plus a closed-loop check when the layout contains a cycle.
- **Ablation** (`bench/ablation_drift.py`): the stitched footprint with correction on vs off, against ground truth footprint and adjacency. Figures go in the report. "Poses as-is" is never the shipped path.

---

## 8. Damage, concealed damage and scope

- **Proposals**: SAM 2 plus colour and texture anomaly cues on each surface texture (rectified into surface coordinates).
- **Classification**: zero-shot classifier or VLM over a closed class list (water stain, mould, crack, peeling or blistering paint, hole or impact, and similar). Evaluate on the staged damage and keep classes honest.
- **Metric extent**: project the mask onto the fitted surface plane to get area and length in metres, with an interval that includes surface-plane and mask-boundary uncertainty.
- **Concealed-damage rules** (`damage/rules/*.yaml`, a transparent rule engine): examples are a ceiling stain under a wet room, which flags possible supply or waste line leakage; a wall-base stain plus baseboard swelling, which flags moisture in the cavity; and mould near an exterior corner, which flags possible envelope leak. Each flag records the **rule ID, the evidence that matched, and the surface**.
- **Scope mapper**: damage class × surface type × extent maps to a line-item catalog (`scope/catalog.yaml`) with quantity and unit, keyed to surface IDs. No pricing is needed, because the brief asks for scope line items.

---

## 9. Hard surfaces and difficult conditions (required by the brief)

| Condition | Detection | Behaviour |
|---|---|---|
| Mirrors | Depth reflection inconsistency, mirror segmentation, symmetric-room-behind-wall artefacts | Mask from plane fitting, flag, inflate local intervals |
| Glass and windows | Missing or erratic LiDAR returns, transparency cues | Treat as opening-like, with provenance flag |
| Wet-look or glossy surfaces | Specular highlights, depth outliers | Down-weight, flag |
| Low light | Exposure and noise statistics | Inflate intervals, warn the operator |
| Motion blur and low texture | Variance of Laplacian, feature counts | Frame rejection, inflation |

`docs/failure_modes.md` lists each with evidence from the benchmark where captured. Deliberately include at least one mirror and one glossy surface case in the benchmark.

---

## 10. Capture route (Route 2 default)

- `capture/PROTOCOL.md`, **one page**, written for a non-engineer. It covers what to install, with exact app names and settings; how to walk (speed, path, height, overlap); how long each tier takes; what to avoid (mirrors in frame, backlit windows, moving people); the photo tier shot list (2-8 stills, corners, doorway shot per connector, ceiling-to-floor coverage); naming and handing files over (folder layout, one command to run); and the device-to-tier mapping.
- **User-test it** on a non-engineer, record their errors in `docs/protocol_usertest.md`, then revise. At the defense they follow it literally.
- The README gets a copy-paste "capture to result" flow that runs in under 15 minutes on a clean machine.
- Route 1 (own iOS app) is explicitly out of scope unless D1 changes.

---

## 11. Benchmark plan (we build it, composition is mandated)

| Set | Contents | Tiers |
|---|---|---|
| B1 | Multi-room: ≥ 3 rooms + connector (hallway or door) | Photo (per-room folders), video, LiDAR |
| B2 | One furnished room with staged damage in ≥ 2 classes | Photo, video, LiDAR |
| B3 | The same rooms (B1 and B2) at all three tiers | included above |
| B4 | At least one room captured **twice at the same tier** (ideally at every tier) | Repeatability |
| B5 | Extra stress captures: mirror, glossy floor, low light | At least video or LiDAR |

- **Ground truth**: laser measurer and tape for wall lengths, ceiling height (several points), opening widths, heights and positions, room-to-room offsets, and damage extents. Use a fixed measurement sheet template (`bench/ground_truth/TEMPLATE.csv`), with repeated measurements to estimate GT noise.
- **Raw sensor data and measurements are submitted**, so store them under `bench/data/` with LFS or a fetch script, plus a manifest with hashes.
- **Staged damage**: removable and reversible. Use washable paint or tea-style stains on a panel, tape for peeling, and a printed or mocked crack. Do not damage anything permanent.
- **No overfitting**: freeze a dev set vs held-out set split in `bench/MANIFEST.md` before tuning. Use the held-out set only for reporting, and the dev set for tuning and calibration (via CV).
- **Gate harness** (`bench/run_benchmark.py`, deterministic): opening detection scored as precision and recall, where a missed opening and a phantom opening both count as misses; ceiling height error and cross-capture spread; wall-length error by tier; repeatability within 1 cm or 0.5% per wall; footprint error and adjacency correctness; non-overlap; interval coverage; timing.
- Output: `reports/benchmark_report.md` with a pass/fail table, repeatability table, head-to-head table and timing.

---

## 12. Deliverables map

| # | Deliverable | Where |
|---|---|---|
| 1 | Compliance matrix (requirement → file → artifact → status) | `docs/compliance_matrix.md` (+ checker) |
| 2 | Capture route and device matrix | `capture/PROTOCOL.md`, `docs/device_matrix.md` |
| 3 | Repo; README to running in < 15 min; one command per capture | `README.md`, `roomscan scan` |
| 4 | Reproduction bundle (regenerate every number from raw inputs; cache replays deterministically and live path also runs) | `scripts/make_repro_bundle.py`, `make repro`, `models/` pinned hashes |
| 5 | Benchmark report | `reports/benchmark_report.md` |
| 6 | Fix loop bundle | `docs/fix_declaration.md`, `reports/fixloop/{before,after}`, `git diff before-fix..after-fix` |
| 7 | Technical report, max 6 pages | `docs/technical_report.md` → PDF (outline in section 16) |
| 8 | Raw benchmark data | `bench/data/`, app exports under `bench/headtohead/` |

Constraints handled: handheld consumer capture only. Every pretrained model, dataset and API is disclosed in `docs/DISCLOSURES.md` (name, version, licence, purpose). Weights are fetched by `scripts/fetch_models.py` with hash pinning. Runtime needs no network and no author-hosted infrastructure.

---

## 13. Process evidence and commit strategy

Rules (also in `.claude/CLAUDE.md`):
1. Commit after every coherent unit of work. Aim for several commits per working session, each building and passing tests.
2. Conventional commit messages (`feat(lidar): fit floor and ceiling planes`, `test: ...`, `docs: ...`, `bench: ...`). Messages explain the **why**.
3. Never squash, never rewrite published history, never backdate. The history has to belong to the person who built it.
4. Commit failures and pivots honestly (`fix`, `revert`, `docs(decisions): drop X because Y`). A real history has false starts.
5. Real data and experiments are committed as they happen (`bench: capture B1 lidar session 1`).
6. Tag milestones: `v0.1-skeleton`, `v0.2-lidar`, `v0.3-video`, `v0.4-photo`, `v0.5-stitch`, `baseline`, `before-fix`, `after-fix`, `final`.
7. No Co-Authored-By or other AI attribution lines in commit messages. AI tools are allowed and expected; disclosure goes in the report and `docs/DISCLOSURES.md`.
8. The author must be able to defend every design decision live with tools closed. Write an ADR in `docs/decisions/NNNN-*.md` for each non-trivial choice, and read and understand each module before moving on.

Expected commit shape: an early plan and compliance-matrix commit, then roughly 5-15 commits per phase in section 14.

---

## 14. Phased roadmap

Each phase ends with passing tests, an updated compliance matrix and a tag.

**Phase 0: Foundations and capture spike**
- Commit the brief and this plan; init the repo (`pyproject.toml`, `uv`, `ruff`, `pytest`, CI skeleton, `.gitignore`, LFS config).
- Capture spike on a real iPhone: test candidate LiDAR apps for export of RGB, depth, pose and intrinsics (D2); test video and photo settings; pick the head-to-head app (D6).
- ADRs: route choice, LiDAR app, stack. `docs/DISCLOSURES.md` stub.

**Phase 1: Contract and vertical slice**
- Pydantic schema → published JSON Schema; example outputs; schema tests.
- CLI with `scan`, `doctor`, `--live/--replay`; pipeline orchestrator with caching, timing and structured logging.
- Stub stages producing valid but trivial output, so end-to-end runs from day one. Plan renderer v0. Compliance matrix v0 plus checker.

**Phase 2: Benchmark capture (starts here, runs in parallel with the code phases)**
- Capture B1, B2, B4 and B5 at all tiers, plus incumbent-app captures for the head-to-head rooms.
- Ground truth sheets, hashes and manifest; dev/held-out split frozen.
- Gate harness skeleton (`bench/gates.py`) in this phase, so metrics exist before the algorithms.

**Phase 3: LiDAR tier**
- Adapter for the chosen export format; gravity alignment; plane fitting; room polygon; ceiling height; area; openings (3D gaps plus 2D detector).
- First gate run on LiDAR. Tag `v0.2-lidar`.

**Phase 4: Video tier**
- Keyframe selection; poses and intrinsics; multi-cue metric scale fusion; reuse of the geometry path. Tag `v0.3-video`.

**Phase 5: Photo tier**
- Per-room folder ingestion; small-baseline multi-view poses; scale fusion; single-room layout; doorway-shot protocol. Tag `v0.4-photo`.

**Phase 6: Stitching and drift**
- Door matching; SE(2) and SE(3) pose graph; loop closure; plane-anchored constraints; overlap checker; **drift ablation** (on vs off) produced as an artifact. Tag `v0.5-stitch`.

**Phase 7: Damage, concealed rules and scope**
- Proposals, classification, metric extent; rule engine with evidence recording; scope catalog and mapper; mirror, glass and low-light hazard handling.

**Phase 8: Uncertainty and calibration**
- Error budget; quality inflation; conformal calibrator per tier; calibration report and reliability plots; "bias vs spread" statement per tier.

**Phase 9: Baseline benchmark and head-to-head**
- Full benchmark run on the held-out set. Tag `baseline`.
- Head-to-head on 2 rooms vs the incumbent app (LiDAR tier, our error vs theirs per dimension; target: beat or tie ≥ 70%). Name the app and version, and commit its exports.

**Phase 10: Fix loop**
1. Identify the single worst gate and its failing number from the baseline.
2. Commit `docs/fix_declaration.md` (one page: worst gate and number; root-cause hypothesis and evidence; intended fix and **predicted number after**). The commit must precede the fix commit.
3. Tag `before-fix`; save the regenerable run under `reports/fixloop/before`.
4. Ship the fix. Re-run; tag `after-fix`; save `reports/fixloop/after`.
5. Write the post-mortem: if the prediction was wrong, say why. Provide a readable diff and `make fixloop`, which regenerates both runs.

**Phase 11: Walk-in hardening**
- `roomscan doctor`: model weights present, versions, disk, RAM, timing estimate.
- Cold-run rehearsals on unseen spaces captured by someone else, with a laser, scored with the same harness. Fix robustness bugs here.
- Time budget and fallbacks (if the primary model fails, fall back to a lighter path with wider intervals). Runbook in `docs/walkin_runbook.md`.

**Phase 12: Packaging and report**
- Repro bundle, fresh-machine test of README in under 15 minutes, compliance matrix final, benchmark report, technical report (max 6 pages), protocol user-test, DISCLOSURES complete. Tag `final`.

---

## 15. Walk-in test readiness (30% of score)

- All three tiers pass a cold-run rehearsal on at least 3 unseen spaces before the defense.
- Only the protocol is followed during the rehearsal, and the run is done by someone other than the author when possible.
- Typical runtime per tier is measured and stated, and the live path works with no network.
- No hard-coded paths, dev-set constants or per-room tuning. A grep check and a test guard against benchmark-specific constants.
- Output always validates against the schema, even for degraded input.
- Defense prep: a one-page "why" for each module and ADR; answer drills on drift, calibration, scale sources, and bias vs spread.

---

## 16. Technical report outline (max 6 pages)

1. Architecture and tier design (diagram), including the device matrix.
2. Scale and geometry: how each tier gets metric scale.
3. Drift handling and the ablation figure.
4. Error budget.
5. Calibration analysis (coverage, bias vs spread per tier).
6. Benchmark and head-to-head results (tables).
7. Fix loop story: prediction vs outcome.
8. Known failure modes (mirrors, glass, low light, and the rest).

---

## 17. Risks and mitigations

| Risk | Mitigation |
|---|---|
| The stock LiDAR app does not export usable pose, depth and intrinsics | Phase 0 spike with two or three candidate apps. ADR records the choice and the fallback. |
| Photo-only metric scale is weak | Multi-cue scale fusion, honest wide intervals, and the ±8% gate. Calibration matters more than precision here. |
| Photo-tier adjacency cannot be inferred | Doorway-shot protocol, door matching, optional hint file, and an explicit statement of which path was used. |
| Compute too slow on the walk-in machine | CPU-feasible models, a time budget, a lighter fallback path, and `doctor`. |
| Overfitting to our rooms | Frozen held-out set, no per-room constants, and cold-run rehearsals on unseen spaces. |
| Physical capture is slow and error-prone | Start capture in Phase 2, capture early, and recapture while ground truth is still fresh. |
| History looks fabricated | Commit continuously and honestly from Phase 0. |
| Cannot defend AI-written code | ADR per decision, module walkthroughs, and understand before moving on. |

---

## 18. Definition of done

- [ ] Three tiers run from one command each and validate against the published schema.
- [ ] Every gate in the brief is measured on the held-out set and reported pass or fail honestly.
- [ ] Drift ablation (on vs off) is in the report; "poses as-is" is not the shipped path.
- [ ] Photo folders produce a stitched, non-overlapping, correctly adjacent plan.
- [ ] Calibration reported per tier; bias vs spread stated.
- [ ] Head-to-head table for 2 rooms with the app name and version; target ≥ 70% beat or tie.
- [ ] Fix declaration committed before the fix; before and after runs regenerable; diff readable.
- [ ] Compliance matrix complete, with all paths verified by the checker.
- [ ] Repro bundle reproduces every reported number; README works on a clean machine in < 15 minutes.
- [ ] Technical report ≤ 6 pages; DISCLOSURES complete; raw data submitted.
- [ ] Cold-run rehearsals on unseen spaces passed for all three tiers.
- [ ] Commit history is continuous and tagged.
