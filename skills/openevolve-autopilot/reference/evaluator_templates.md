# Evaluator Templates

Use these templates as starting points when generating evaluators for different optimization goals.

## Table of Contents
1. [Performance/Speed Optimization](#performance-speed)
2. [Correctness/Accuracy Optimization](#correctness-accuracy)
3. [Memory Optimization](#memory)
4. [Multi-Objective (Speed + Correctness)](#multi-objective)
5. [Algorithm Quality](#algorithm-quality)
6. [Code Size / Simplicity](#code-simplicity)

---

## Performance/Speed Optimization {#performance-speed}

```python
"""Evaluator for speed optimization."""
import importlib.util
import time
import sys
import os

# --- Test data (customize per target) ---
TEST_INPUTS = [
    # Add representative inputs here
]

EXPECTED_OUTPUTS = [
    # Corresponding expected outputs for correctness checking
]

NUM_TIMING_TRIALS = 5
TIMEOUT_SECONDS = 30


def evaluate(program_path: str) -> dict:
    """Evaluate evolved program for speed while maintaining correctness."""
    spec = importlib.util.spec_from_file_location("evolved", program_path)
    if spec is None or spec.loader is None:
        return {"combined_score": 0.0, "error": "Failed to load module"}

    module = importlib.util.module_from_spec(spec)
    try:
        spec.loader.exec_module(module)
    except Exception as e:
        return {"combined_score": 0.0, "error": f"Import error: {e}"}

    target_func = getattr(module, "TARGET_FUNCTION_NAME", None)
    if target_func is None:
        return {"combined_score": 0.0, "error": "Target function not found"}

    # Step 1: Correctness check
    correct = 0
    total = len(TEST_INPUTS)
    for inp, expected in zip(TEST_INPUTS, EXPECTED_OUTPUTS):
        try:
            result = target_func(inp)
            if result == expected:  # Or use np.allclose for numerical results
                correct += 1
        except Exception:
            pass

    correctness_score = correct / total if total > 0 else 0.0
    if correctness_score < 0.5:
        return {
            "combined_score": 0.0,
            "correctness": correctness_score,
            "error": "Too many incorrect results",
        }

    # Step 2: Speed measurement
    times = []
    for _ in range(NUM_TIMING_TRIALS):
        start = time.perf_counter()
        try:
            for inp in TEST_INPUTS:
                target_func(inp)
        except Exception:
            return {"combined_score": 0.0, "correctness": correctness_score, "error": "Runtime crash"}
        elapsed = time.perf_counter() - start
        times.append(elapsed)

    avg_time = sum(times) / len(times)
    # Normalize: faster is better. Baseline time should be set from initial program.
    BASELINE_TIME = 1.0  # Set this from the initial program's timing
    speed_score = min(BASELINE_TIME / (avg_time + 1e-9), 5.0) / 5.0  # Cap at 5x improvement

    combined_score = speed_score * correctness_score

    return {
        "combined_score": combined_score,
        "speed_score": speed_score,
        "correctness": correctness_score,
        "avg_time_seconds": avg_time,
        "min_time_seconds": min(times),
    }


def evaluate_stage1(program_path: str) -> dict:
    """Quick check: does it run and produce correct output for one case?"""
    spec = importlib.util.spec_from_file_location("evolved", program_path)
    if spec is None or spec.loader is None:
        return {"combined_score": 0.0}
    module = importlib.util.module_from_spec(spec)
    try:
        spec.loader.exec_module(module)
    except Exception:
        return {"combined_score": 0.0}

    target_func = getattr(module, "TARGET_FUNCTION_NAME", None)
    if target_func is None:
        return {"combined_score": 0.0}

    try:
        result = target_func(TEST_INPUTS[0])
        if result == EXPECTED_OUTPUTS[0]:
            return {"combined_score": 1.0}
        return {"combined_score": 0.0}
    except Exception:
        return {"combined_score": 0.0}
```

---

## Correctness/Accuracy Optimization {#correctness-accuracy}

```python
"""Evaluator for correctness optimization."""
import importlib.util
import math

TEST_CASES = [
    # (input, expected_output) tuples
]


def evaluate(program_path: str) -> dict:
    """Evaluate evolved program for correctness across test cases."""
    spec = importlib.util.spec_from_file_location("evolved", program_path)
    if spec is None or spec.loader is None:
        return {"combined_score": 0.0, "error": "Failed to load module"}

    module = importlib.util.module_from_spec(spec)
    try:
        spec.loader.exec_module(module)
    except Exception as e:
        return {"combined_score": 0.0, "error": f"Import error: {e}"}

    target_func = getattr(module, "TARGET_FUNCTION_NAME", None)
    if target_func is None:
        return {"combined_score": 0.0, "error": "Target function not found"}

    passed = 0
    total = len(TEST_CASES)
    errors = []
    for i, (inp, expected) in enumerate(TEST_CASES):
        try:
            result = target_func(inp)
            if _check_equal(result, expected):
                passed += 1
            else:
                errors.append(f"Case {i}: expected {expected}, got {result}")
        except Exception as e:
            errors.append(f"Case {i}: {e}")

    score = passed / total if total > 0 else 0.0

    return {
        "combined_score": score,
        "pass_rate": score,
        "passed": passed,
        "total": total,
        "errors": errors[:5],  # Limit error output
    }


def _check_equal(result, expected, tol=1e-6):
    """Flexible equality check supporting floats, lists, etc."""
    if isinstance(expected, float):
        return abs(result - expected) < tol
    if isinstance(expected, (list, tuple)):
        if len(result) != len(expected):
            return False
        return all(_check_equal(r, e, tol) for r, e in zip(result, expected))
    return result == expected
```

---

## Memory Optimization {#memory}

```python
"""Evaluator for memory usage optimization."""
import importlib.util
import tracemalloc


def evaluate(program_path: str) -> dict:
    """Evaluate evolved program for memory efficiency."""
    spec = importlib.util.spec_from_file_location("evolved", program_path)
    if spec is None or spec.loader is None:
        return {"combined_score": 0.0, "error": "Failed to load module"}

    module = importlib.util.module_from_spec(spec)
    try:
        spec.loader.exec_module(module)
    except Exception as e:
        return {"combined_score": 0.0, "error": f"Import error: {e}"}

    target_func = getattr(module, "TARGET_FUNCTION_NAME", None)
    if target_func is None:
        return {"combined_score": 0.0, "error": "Target function not found"}

    # Correctness check first
    try:
        result = target_func(TEST_INPUT)
        if result != EXPECTED_OUTPUT:
            return {"combined_score": 0.0, "error": "Incorrect output"}
    except Exception as e:
        return {"combined_score": 0.0, "error": str(e)}

    # Memory measurement
    tracemalloc.start()
    try:
        target_func(TEST_INPUT)
    except Exception as e:
        tracemalloc.stop()
        return {"combined_score": 0.0, "error": str(e)}

    current, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()

    peak_mb = peak / 1024 / 1024
    BASELINE_PEAK_MB = 10.0  # Set from initial program
    memory_score = min(BASELINE_PEAK_MB / (peak_mb + 1e-9), 5.0) / 5.0

    return {
        "combined_score": memory_score,
        "peak_memory_mb": peak_mb,
        "current_memory_mb": current / 1024 / 1024,
    }
```

---

## Multi-Objective (Speed + Correctness) {#multi-objective}

```python
"""Evaluator for balanced speed and correctness optimization."""
import importlib.util
import time


def evaluate(program_path: str) -> dict:
    spec = importlib.util.spec_from_file_location("evolved", program_path)
    if spec is None or spec.loader is None:
        return {"combined_score": 0.0, "error": "Failed to load module"}

    module = importlib.util.module_from_spec(spec)
    try:
        spec.loader.exec_module(module)
    except Exception as e:
        return {"combined_score": 0.0, "error": f"Import error: {e}"}

    target_func = getattr(module, "TARGET_FUNCTION_NAME", None)
    if target_func is None:
        return {"combined_score": 0.0, "error": "Target function not found"}

    # Correctness
    correct = 0
    for inp, expected in TEST_CASES:
        try:
            if target_func(inp) == expected:
                correct += 1
        except Exception:
            pass
    correctness = correct / len(TEST_CASES)

    # Gate: must be at least 80% correct
    if correctness < 0.8:
        return {"combined_score": correctness * 0.5, "correctness": correctness, "speed_score": 0.0}

    # Speed (only measure if correctness passes)
    times = []
    for _ in range(3):
        start = time.perf_counter()
        for inp, _ in TEST_CASES:
            target_func(inp)
        times.append(time.perf_counter() - start)

    avg_time = sum(times) / len(times)
    BASELINE_TIME = 1.0
    speed_score = min(BASELINE_TIME / (avg_time + 1e-9), 5.0) / 5.0

    # Weighted combination: 60% correctness, 40% speed
    combined = 0.6 * correctness + 0.4 * speed_score

    return {
        "combined_score": combined,
        "correctness": correctness,
        "speed_score": speed_score,
        "avg_time": avg_time,
    }
```

---

## Algorithm Quality {#algorithm-quality}

For optimizing algorithmic solutions (e.g., optimization algorithms, search algorithms):

```python
"""Evaluator for algorithm quality (solution optimality)."""
import importlib.util
import random
import numpy as np

# Fix random seed for reproducibility
random.seed(42)
np.random.seed(42)

# Generate test problems
TEST_PROBLEMS = [
    # Define problem instances here
]

KNOWN_OPTIMA = [
    # Known optimal solutions (or best known) for comparison
]


def evaluate(program_path: str) -> dict:
    spec = importlib.util.spec_from_file_location("evolved", program_path)
    if spec is None or spec.loader is None:
        return {"combined_score": 0.0, "error": "Failed to load module"}

    module = importlib.util.module_from_spec(spec)
    try:
        spec.loader.exec_module(module)
    except Exception as e:
        return {"combined_score": 0.0, "error": f"Import error: {e}"}

    solve_func = getattr(module, "solve", None) or getattr(module, "run", None)
    if solve_func is None:
        return {"combined_score": 0.0, "error": "No solve/run function found"}

    quality_scores = []
    reliability_count = 0
    total_runs = len(TEST_PROBLEMS) * 3  # 3 trials per problem

    for problem, optimum in zip(TEST_PROBLEMS, KNOWN_OPTIMA):
        for trial in range(3):
            try:
                result = solve_func(problem)
                # Compute quality relative to known optimum
                quality = _compute_quality(result, optimum)
                quality_scores.append(quality)
                if quality > 0.9:
                    reliability_count += 1
            except Exception:
                quality_scores.append(0.0)

    avg_quality = np.mean(quality_scores) if quality_scores else 0.0
    reliability = reliability_count / total_runs

    combined = 0.7 * avg_quality + 0.3 * reliability

    return {
        "combined_score": combined,
        "avg_quality": float(avg_quality),
        "reliability": reliability,
        "best_quality": float(max(quality_scores)) if quality_scores else 0.0,
    }


def _compute_quality(result, optimum):
    """Compute quality of result relative to known optimum. Returns 0-1."""
    # Customize based on the problem type
    # For minimization: quality = optimum / (result + epsilon)
    # For maximization: quality = result / (optimum + epsilon)
    pass
```

---

## Code Size / Simplicity {#code-simplicity}

```python
"""Evaluator that rewards correct AND concise code."""
import importlib.util
import ast


def evaluate(program_path: str) -> dict:
    with open(program_path, "r") as f:
        code = f.read()

    # Correctness first
    spec = importlib.util.spec_from_file_location("evolved", program_path)
    module = importlib.util.module_from_spec(spec)
    try:
        spec.loader.exec_module(module)
    except Exception as e:
        return {"combined_score": 0.0, "error": str(e)}

    target_func = getattr(module, "TARGET_FUNCTION_NAME", None)
    if target_func is None:
        return {"combined_score": 0.0, "error": "Function not found"}

    correct = sum(1 for inp, exp in TEST_CASES if _safe_call(target_func, inp) == exp)
    correctness = correct / len(TEST_CASES)

    if correctness < 0.9:
        return {"combined_score": correctness * 0.3, "correctness": correctness}

    # Code complexity metrics
    try:
        tree = ast.parse(code)
    except SyntaxError:
        return {"combined_score": 0.0, "error": "Syntax error"}

    num_lines = len([l for l in code.split("\n") if l.strip() and not l.strip().startswith("#")])
    num_nodes = sum(1 for _ in ast.walk(tree))

    BASELINE_LINES = 50  # Set from initial program
    simplicity = min(BASELINE_LINES / (num_lines + 1), 2.0) / 2.0

    combined = 0.7 * correctness + 0.3 * simplicity

    return {
        "combined_score": combined,
        "correctness": correctness,
        "simplicity": simplicity,
        "num_lines": num_lines,
        "ast_nodes": num_nodes,
    }


def _safe_call(func, inp):
    try:
        return func(inp)
    except Exception:
        return None
```
