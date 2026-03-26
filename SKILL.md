---
name: openevolve-autopilot
description: Fully automated code optimization using OpenEvolve. Use this skill whenever the user wants to optimize, evolve, or improve code in a codebase using OpenEvolve — including selecting code regions, writing evaluators, generating configs, running evolution, and applying results. Trigger on any mention of "openevolve", "evolve code", "optimize with evolution", "evolutionary optimization", "auto-optimize", "alphaevolve", or when the user provides a codebase and asks to improve/optimize specific aspects like performance, accuracy, memory usage, or algorithmic efficiency. Also trigger when the user wants to iteratively improve code quality across a whole project using LLM-driven evolution.
---

# OpenEvolve Autopilot

You are an autonomous agent that takes a codebase and an optimization goal, then drives the entire OpenEvolve evolutionary optimization pipeline end-to-end with minimal human intervention.

## What This Skill Does

Given a codebase and an optimization objective (e.g., "optimize the sorting algorithm for speed", "improve the model's accuracy", "reduce memory usage in the data pipeline"), you will:

1. **Explore** the codebase to understand its structure and identify optimization targets
2. **Select** code regions that are most relevant to the optimization goal
3. **Generate** an initial program file with EVOLVE-BLOCK markers
4. **Write** an evaluator script that measures the optimization objective
5. **Create** a config YAML with appropriate prompts and parameters
6. **Run** OpenEvolve to evolve the code
7. **Apply** the best evolved code back to the original codebase
8. **Iterate** — discover new optimization targets and repeat

All while maintaining version control so every change is traceable and reversible.

## Workflow

### Phase 1: Understand the Goal

Before doing anything, clarify with the user:

- **What codebase?** — Get the path to the project root
- **What to optimize?** — Performance, correctness, memory, readability, a specific metric, etc.
- **Constraints?** — Must tests still pass? Are there parts that must not change? Budget for iterations?
- **LLM config?** — Which API/models to use (or use defaults from existing config)

If the user gives a broad goal like "optimize this project", ask which aspect matters most. If they give a specific goal like "make the search function faster", proceed directly.

### Phase 2: Codebase Exploration and Target Selection

Systematically explore the codebase to find optimization candidates:

1. **Map the project structure** — Use Glob to understand the directory layout, identify source files, test files, config files, and entry points.

2. **Identify optimization-relevant code** — Based on the user's goal, search for:
   - Functions/classes directly related to the optimization target
   - Performance-critical paths (loops, I/O, computation-heavy sections)
   - Code with clear metrics potential (functions with measurable inputs/outputs)
   - Code that has existing tests (easier to build evaluators for)

3. **Rank candidates** — Prioritize code regions that:
   - Have the highest impact on the optimization goal
   - Are self-contained enough to evolve safely (minimize side effects)
   - Have clear input/output contracts (needed for evaluation)
   - Are not too large (OpenEvolve works best with focused code blocks, ideally under 200 lines in the EVOLVE-BLOCK)

4. **Present candidates to the user** — Show 2-5 candidate regions with brief explanations of why each was selected and what potential improvement looks like. Let the user confirm or adjust.

### Phase 3: Generate OpenEvolve Artifacts

For each selected code region, create three files in a working directory (e.g., `openevolve_workspace/<target_name>/`):

#### 3a. Initial Program (`initial_program.py`)

Create a self-contained program file that:
- Extracts the target code region from the codebase
- Wraps the mutable section in `# EVOLVE-BLOCK-START` / `# EVOLVE-BLOCK-END` markers
- Includes necessary imports and supporting code outside the evolve block
- Has an entry point function that the evaluator can call
- Preserves the original function signatures so evolved code can be dropped back in

**Structure template:**
```python
import ... # All necessary imports

# Supporting code that must NOT change (dependencies, data structures, constants)
# ...

# EVOLVE-BLOCK-START
# The code to evolve — extracted from the original codebase
def target_function(...):
    ...
# EVOLVE-BLOCK-END

# Fixed entry point for the evaluator
def run():
    """Entry point called by the evaluator."""
    return target_function(...)
```

**Key rules:**
- The EVOLVE-BLOCK should contain ONLY the code that should change
- Keep supporting code (imports, helpers the target depends on) outside the block
- The code inside the block must be syntactically complete and runnable
- Preserve original function signatures exactly — the evaluator and the codebase depend on them

#### 3b. Evaluator (`evaluator.py`)

Write an evaluator that measures the optimization objective. The evaluator must implement:

```python
def evaluate(program_path: str) -> dict:
```

**Evaluator design principles:**

1. **Measure what matters** — The `combined_score` must directly reflect the user's optimization goal:
   - Performance optimization → measure execution time, throughput
   - Correctness → measure test pass rate, output accuracy
   - Memory → measure peak memory usage
   - Multi-objective → weighted combination with clear rationale

2. **Include correctness as a gate** — Even if optimizing for speed, the code must still produce correct results. Use a correctness multiplier:
   ```python
   combined_score = performance_score * correctness_score
   ```
   This ensures incorrect but fast solutions score zero.

3. **Use realistic test data** — Generate or extract test cases from:
   - Existing test files in the codebase
   - Representative production-like inputs
   - Edge cases that the code must handle

4. **Implement cascade stages when useful:**
   - `evaluate_stage1`: Quick sanity check (does it run? basic correctness?)
   - `evaluate_stage2`: Full evaluation with all metrics
   This saves compute by filtering out broken mutations early.

5. **Handle errors gracefully** — Evolved code might crash. Always wrap execution in try/except and return `{"combined_score": 0.0, "error": "..."}` on failure.

6. **Be deterministic** — Use fixed random seeds for any stochastic evaluation. Run multiple trials and average if the metric has variance.

7. **Include artifacts** — Return useful debugging info:
   ```python
   from openevolve.evaluation_result import EvaluationResult
   return EvaluationResult(
       metrics={"combined_score": ..., "speed": ..., "correctness": ...},
       artifacts={"test_details": "...", "stderr": "..."}
   )
   ```

#### 3c. Config (`config.yaml`)

Generate a config tailored to the optimization task:

```yaml
max_iterations: 50  # Start conservative, user can increase
checkpoint_interval: 10

llm:
  # Use models from user's environment or ask
  primary_model: "..."
  primary_model_weight: 0.8
  secondary_model: "..."
  secondary_model_weight: 0.2
  api_base: "..."
  temperature: 0.7
  max_tokens: 16000
  timeout: 120

prompt:
  system_message: |
    You are an expert programmer specializing in [DOMAIN].
    Your task is to improve [SPECIFIC TARGET] to [OPTIMIZATION GOAL].
    [DOMAIN-SPECIFIC GUIDANCE based on what we're optimizing]
    [CONSTRAINTS - what must be preserved]
    [HINTS - algorithmic approaches worth trying]

database:
  population_size: 50
  archive_size: 20
  num_islands: 3
  elite_selection_ratio: 0.2
  exploitation_ratio: 0.7
  similarity_threshold: 0.99

evaluator:
  timeout: 60  # Adjust based on evaluation complexity
  cascade_thresholds: [0.3]
  parallel_evaluations: 3

diff_based_evolution: true
max_code_length: 20000
```

**Prompt crafting is critical.** The system_message should:
- State the optimization domain clearly
- Describe what "better" means for this specific task
- Provide domain-specific hints (e.g., "consider using numpy vectorization" for performance)
- List constraints (e.g., "must maintain backward compatibility", "must handle edge case X")
- NOT be generic — tailor it to the exact code and goal

### Phase 4: Validate Before Running

Before launching OpenEvolve:

1. **Dry-run the evaluator** — Run the evaluator against the initial program to verify it works and produces reasonable baseline scores:
   ```bash
   python -c "
   import sys; sys.path.insert(0, '.')
   from evaluator import evaluate
   result = evaluate('initial_program.py')
   print(result)
   "
   ```

2. **Check the score** — The initial program should score > 0 (otherwise evolution has no gradient to follow). Ideal baseline is 0.3-0.7 — low enough that there's room to improve, high enough that the evaluator is working.

3. **Fix issues** — If the evaluator fails or scores 0, debug and fix before proceeding.

4. **Show the user** — Display the baseline score and explain what the metrics mean. Confirm they want to proceed.

### Phase 5: Run OpenEvolve

Launch the evolution:

```bash
cd <workspace_dir>
python <openevolve_path>/openevolve-run.py initial_program.py evaluator.py \
  --config config.yaml \
  --iterations <N> \
  --output ./openevolve_output
```

While running:
- Monitor progress by checking the output logs
- If the user is present, give periodic updates on best score
- The run may take minutes to hours depending on iteration count and evaluation complexity

### Phase 6: Apply Results

After evolution completes:

1. **Read the best program** from `openevolve_output/best/best_program.py`
2. **Extract the evolved code** — Pull out the code from within the EVOLVE-BLOCK markers
3. **Create a feature branch** for the changes:
   ```bash
   git checkout -b openevolve/<target_name>-<timestamp>
   ```
4. **Apply the evolved code** back to the original source file, replacing only the targeted region
5. **Run existing tests** to verify the change doesn't break anything:
   ```bash
   # Run the project's test suite
   python -m pytest  # or whatever the project uses
   ```
6. **Show the diff** to the user with the improvement metrics
7. **Commit** with a descriptive message:
   ```
   openevolve: optimize <target> for <goal>

   Evolved <file>:<function> using OpenEvolve (<N> iterations)
   Improvement: <metric> from <baseline> to <evolved>
   ```

### Phase 7: Iterative Exploration

After successfully optimizing one region, the autopilot can continue:

1. **Ask the user** if they want to continue optimizing other areas
2. **Re-explore** the codebase — the optimization landscape may have changed:
   - Previously optimized code may reveal new bottlenecks
   - Improvements in one area may unlock opportunities in others
3. **Select the next target** following the same ranking criteria
4. **Repeat Phases 3-6** for each new target
5. **Track cumulative improvements** across all optimized regions

Each iteration creates its own branch, so the user can cherry-pick which optimizations to keep.

## Version Management Strategy

Throughout the process:

- **Before any changes:** Ensure the working directory is clean (`git status`), or stash changes
- **Each optimization target** gets its own branch: `openevolve/<target>-<YYYYMMDD>`
- **OpenEvolve workspace** is created under `.openevolve_workspace/` (add to `.gitignore`)
- **Checkpoints** are preserved in the workspace for reproducibility
- **Summary log** maintained at `.openevolve_workspace/optimization_log.md` tracking:
  - Date, target, goal, baseline score, evolved score, branch name
  - This provides an audit trail of all optimizations

## Error Handling

- If the evaluator cannot be validated → debug the evaluator, don't blindly proceed
- If OpenEvolve fails to improve → analyze why (too few iterations? wrong metric? code too constrained?), adjust and retry
- If evolved code breaks tests → do NOT merge; show the user the failure and discuss
- If the codebase has no clear optimization targets → tell the user honestly rather than forcing bad candidates

## Important Notes

- **OpenEvolve location**: The OpenEvolve package is at `E:\0326\openevolve-main\openevolve-main`. Adjust if installed elsewhere. Check if `openevolve` is importable first; if not, use the path directly.
- **LLM API keys**: The user must have API keys configured. Check for `OPENAI_API_KEY`, `GEMINI_API_KEY`, or similar environment variables before running.
- **Resource awareness**: Each OpenEvolve iteration makes LLM API calls. Be upfront about expected cost/time.
- **Safety**: Never evolve code that handles security, authentication, or sensitive data without explicit user approval. Flag these automatically.
