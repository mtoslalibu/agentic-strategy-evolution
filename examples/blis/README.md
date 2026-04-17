# Running Nous on BLIS

This example shows how to run a single Nous iteration on [BLIS](https://github.com/inference-sim/inference-sim), a discrete-event simulator for LLM inference serving systems.

## Prerequisites

- Python 3.11+
- An LLM API key set as an environment variable (e.g., `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or any [LiteLLM-supported provider](https://docs.litellm.ai/docs/providers))
- Nous installed: `pip install -e ".[dev]"`

## Campaign configuration

The `campaign.yaml` in this directory configures Nous for BLIS:

| Section | What it controls |
|---------|-----------------|
| `target_system.name` | Human-readable name shown in prompts |
| `target_system.description` | System description given to all agents |
| `target_system.observable_metrics` | What agents can measure (TTFT, TPOT, throughput, etc.) |
| `target_system.controllable_knobs` | What agents can change (scheduler_policy, max_batch_size, etc.) |
| `review.design_perspectives` | Reviewer perspectives for hypothesis bundle review (5 perspectives) |
| `review.findings_perspectives` | Reviewer perspectives for findings review (10 perspectives) |
| `review.max_review_rounds` | Maximum convergence rounds per review gate |

## Running a single iteration

```python
import json
import shutil
from pathlib import Path

import yaml

from orchestrator.engine import Engine
from orchestrator.llm_dispatch import LLMDispatcher
from orchestrator.gates import HumanGate

# 1. Set up working directory
work_dir = Path("blis-run-001")
work_dir.mkdir(exist_ok=True)

# Copy templates
for t in ["state.json", "ledger.json", "principles.json"]:
    shutil.copy(f"templates/{t}", work_dir / t)

# Update run ID
state = json.loads((work_dir / "state.json").read_text())
state["run_id"] = "blis-run-001"
(work_dir / "state.json").write_text(json.dumps(state, indent=2))

# 2. Load campaign config
campaign = yaml.safe_load(Path("examples/blis/campaign.yaml").read_text())

# 3. Initialize components
engine = Engine(work_dir)
dispatcher = LLMDispatcher(work_dir=work_dir, campaign=campaign)
gate = HumanGate()  # interactive approval prompts

# 4. Run the loop
iteration = 1
iter_dir = work_dir / "runs" / f"iter-{iteration}"

# INIT -> FRAMING: produce problem.md
engine.transition("FRAMING")
dispatcher.dispatch(
    "planner", "frame",
    output_path=iter_dir / "problem.md", iteration=iteration,
)

# FRAMING -> DESIGN: produce hypothesis bundle
engine.transition("DESIGN")
dispatcher.dispatch(
    "planner", "design",
    output_path=iter_dir / "bundle.yaml", iteration=iteration,
)

# DESIGN -> DESIGN_REVIEW: reviewers evaluate the bundle
engine.transition("DESIGN_REVIEW")
for perspective in campaign["review"]["design_perspectives"]:
    dispatcher.dispatch(
        "reviewer", "review-design",
        output_path=iter_dir / "reviews" / f"review-{perspective}.md",
        iteration=iteration, perspective=perspective,
    )

# DESIGN_REVIEW -> HUMAN_DESIGN_GATE: you approve or reject
engine.transition("HUMAN_DESIGN_GATE")
gate.prompt(
    "Review the bundle and reviews. Approve?",
    artifact_path=str(iter_dir / "bundle.yaml"),
    reviews=[str(p) for p in (iter_dir / "reviews").glob("review-*.md")],
)

# HUMAN_DESIGN_GATE -> RUNNING: executor analyzes the system
engine.transition("RUNNING")
dispatcher.dispatch(
    "executor", "run",
    output_path=iter_dir / "findings.json", iteration=iteration,
)

# RUNNING -> FINDINGS_REVIEW: reviewers evaluate findings
engine.transition("FINDINGS_REVIEW")
for perspective in campaign["review"]["findings_perspectives"]:
    dispatcher.dispatch(
        "reviewer", "review-findings",
        output_path=iter_dir / "reviews" / f"review-findings-{perspective}.md",
        iteration=iteration, perspective=perspective,
    )

# FINDINGS_REVIEW -> HUMAN_FINDINGS_GATE
engine.transition("HUMAN_FINDINGS_GATE")
gate.prompt("Review the findings and reviews. Approve?")

# HUMAN_FINDINGS_GATE -> TUNING -> EXTRACTION
engine.transition("TUNING")
engine.transition("EXTRACTION")
dispatcher.dispatch(
    "extractor", "extract",
    output_path=work_dir / "principles.json", iteration=iteration,
)

# EXTRACTION -> DONE
engine.transition("DONE")
print("Iteration complete! Check", iter_dir)
```

## Expected output

After running, your working directory will contain:

```
blis-run-001/
  state.json              # phase: DONE
  principles.json         # extracted principles
  ledger.json
  runs/
    iter-1/
      problem.md          # problem framing
      bundle.yaml         # hypothesis bundle
      findings.json       # executor findings
      reviews/
        review-*.md       # design reviews
        review-findings-*.md  # findings reviews
```

## Phase 2 limitation

In Phase 2, the executor operates in **analysis mode** — it reasons about the BLIS codebase and mechanisms but does not run actual benchmarks. The executor produces findings based on its understanding of how scheduling policies, batching, and cache management affect performance.

Phase 3 will add real experiment execution via shell access.

## Customizing

To adapt this for a different LLM inference system:

1. Copy `campaign.yaml` to a new directory
2. Update `target_system` fields (name, description, metrics, knobs)
3. Optionally adjust reviewer perspectives in `review`
4. Run the same Python script with the new campaign config
