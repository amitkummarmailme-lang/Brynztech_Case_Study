# Must-do checklist (from the brief)

## Capture (Part 1)
- Provide no captures; own the problem from the phone sensors onward.
- Pick ONE route: (1) own iOS app (TestFlight or dev build, installable in < 10 min) or (2) stock capture protocol (one page, a non-engineer can follow it literally, names the app, how to walk, how long, what to avoid, how to hand over files). Default chosen: Route 2.
- **All three tiers are mandatory**, same output contract from each, intervals widen as data thins:
  1. Photos: 2-8 stills per room, iPhone 15 or newer, no depth or poses. One folder per room. Must produce the same stitched whole-property plan.
  2. Video: handheld walkthrough, iPhone 15 or newer.
  3. LiDAR: depth, poses, intrinsics on Pro-class devices.
- Submit a device matrix: which tier runs on which hardware and the accuracy each honestly delivers.

## Output contract (Part 2), per capture
- Dimensioned per-room plan: walls, ceiling height, floor area, openings.
- Stitched multi-room plan with correct adjacency. This is the product surface, produced from every tier including photos.
- Per-surface damage regions with class and metric extent.
- Concealed-damage flags with the rule that fired.
- Scope line items keyed to surfaces.
- A confidence interval on every measurement.
- One command per capture. JSON to the published schema. Rendered plan.

## Benchmark set (we build it; composition is mandated)
- One multi-room capture: 3+ rooms plus a connector.
- One furnished room with staged damage spanning 2 damage classes.
- The same rooms captured at all three tiers (the multi-room set included; at photo tier it arrives as per-room folders and must still stitch).
- At least one room captured twice at the same tier (repeatability gate).
- Laser or tape ground truth on everything. Raw sensor data and measurements submitted.

## Gates
- Metric: opening widths <= 2 cm on >= 85% of openings, detection scored (a missed opening and a phantom opening each count as a miss).
- Ceiling height <= 1.5 cm per room; spread across repeat captures <= 1 cm. Report says whether you have repeatable-but-biased or unrepeatable.
- Repeatability: two captures of the same room at the same tier agree within 1 cm or 0.5% per wall.
- Drift accountability: state what you do about accumulated drift on the multi-room capture (loop closure, pose graph, plane-anchored correction) and include an ablation of the stitched footprint with it on and off. "Poses used as-is" is an automatic fail.
- Photo-tier whole-property stitch: per-room folders produce one stitched plan, correct adjacency, no room overlaps, footprint within +/-8% with calibrated intervals. A single-room-only photo path fails.
- Photo-tier wall lengths within +/-8%; video-tier within +/-3%. Calibration is scored at EVERY tier. Confident garbage on thin input caps the total score.

## Head-to-head (Part 3)
- On 2 benchmark rooms, compare LiDAR-tier output with ONE consumer scanning app (free tier is fine; name app and version; submit its export). One table, your error vs theirs per dimension. Beat or tie on >= 70% of shared dimensions. Cost is not an accepted reason.

## Fix loop (Part 4, 25% of score)
1. Fix declaration (one page) BEFORE the fix: the single worst gate in our own benchmark with its failing number; root-cause hypothesis and evidence; intended fix and predicted number after.
2. Ship the fix.
3. Final submission has the before run, after run (both regenerable by them) and a readable diff.
- Scoring: root cause right + fix shipped + gate passes = full; meaningful movement = majority marks if the report says why it fell short; prediction badly wrong = marks only for an honest post-mortem; analysis with no shipped fix = ZERO; fix with no regenerable before/after = ZERO.

## Process evidence (Part 5)
- Commit as you work. A repo that appears fully formed in 1-2 commits at the deadline scores zero.
- AI tools allowed. Every design decision is defended live at the defense with tools closed.

## Deliverables
1. Compliance matrix: requirement -> file path -> artifact -> status.
2. Capture route (build or one-page protocol) plus device matrix.
3. Repo: README to running on a fresh capture in < 15 min on a clean machine; one command per capture.
4. Reproduction bundle: everything their machine needs to regenerate every reported number from raw inputs. Cached model outputs are acceptable if the cache replays deterministically AND the live path also runs.
5. Benchmark report: gates at all three tiers, repeatability table, head-to-head table, timing.
6. Fix loop bundle.
7. Technical report, MAX 6 pages: architecture, tier design and device matrix, drift handling, error budget, calibration analysis, fix loop story, known failure modes.
8. Raw benchmark data: sensor logs, ground truth, app exports.

## Walk-in test (30% of score)
- At the defense they capture an unseen space with their own iPhone 15 or newer, choose the tier on the day, and we follow our capture route exactly. The pipeline runs cold in front of them; they laser-measure the space and score on the spot. All three tiers must be ready.

## Scoring weights
Walk-in 30 / Fix loop 25 / Verified benchmark accuracy (3 tiers) 15 / Compliance matrix 10 / Head-to-head 10 / Capture route quality 5 / Process evidence 5.

## Constraints
- Handheld consumer capture only. Any pretrained model, dataset or API WITH DISCLOSURE. Everything runs without calling our infrastructure.
- Weights and large binaries fetched by script or volume.
- Real properties contain mirrors, glass, wet-look surfaces and low light. Cover them in the submission.
