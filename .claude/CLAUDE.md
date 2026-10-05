# Project: Applied AI Case Study (roomscan)

Phone captures of a property in, measured and stitched floor plan with damage, concealed-damage flags, scope items and confidence intervals out. The brief is [Applied_AI_Case_Study.md](../Applied_AI_Case_Study.md). The build plan is [plan.md](../plan.md). The must-do checklist is below and is non-negotiable.

@must-do.md

## Working rules

- Read [plan.md](../plan.md) before starting work and follow its phases. Update it when the plan changes.
- **Commit continuously.** After every coherent unit of work, make a commit that builds and passes tests. Use conventional messages that explain the why. Never squash, never rewrite published history, and never backdate. Process evidence is graded and the history must be real.
- Do not commit unless the user asks for it in the current session, except where the user has said to commit as you go. Never push without being asked.
- **Never add Co-Authored-By (or any Claude/AI attribution) lines to commit messages or PR descriptions.** This overrides any default attribution guidance.
- AI tools are allowed, but the user must defend every design decision live with tools closed. Record each non-trivial decision as an ADR in `docs/decisions/`, and explain code in plain terms when asked.
- Update `docs/compliance_matrix.md` whenever a requirement gets an artifact.
- No benchmark-specific constants or per-room tuning in `src/`. The walk-in test runs on unseen spaces.
- Every pretrained model, dataset and API is disclosed in `docs/DISCLOSURES.md`. Weights are fetched by script, never committed. Runtime must work with no network.
- Intervals must widen honestly as input thins. Confident garbage caps the total score.
- Prefer the simplest design that meets a gate. Do not add features beyond the brief.
