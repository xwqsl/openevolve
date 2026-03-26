# Config Templates

Pre-built config snippets for common optimization scenarios. Mix and match sections as needed.

## Quick Start Config (Conservative)

For first-time use or exploration. Low iteration count, safe defaults.

```yaml
max_iterations: 30
checkpoint_interval: 10

llm:
  temperature: 0.7
  max_tokens: 16000
  timeout: 120

database:
  population_size: 30
  archive_size: 15
  num_islands: 2
  elite_selection_ratio: 0.2
  exploitation_ratio: 0.7

evaluator:
  timeout: 60
  cascade_thresholds: [0.3]
  parallel_evaluations: 2

diff_based_evolution: true
max_code_length: 15000
```

## Production Config (Thorough)

For when you want serious optimization and are willing to spend more compute.

```yaml
max_iterations: 200
checkpoint_interval: 20

llm:
  temperature: 0.8
  max_tokens: 32000
  timeout: 180

database:
  population_size: 100
  archive_size: 40
  num_islands: 5
  elite_selection_ratio: 0.15
  exploitation_ratio: 0.6
  migration_interval: 30
  migration_rate: 0.15

evaluator:
  timeout: 120
  cascade_evaluation: true
  cascade_thresholds: [0.3, 0.6]
  parallel_evaluations: 4

diff_based_evolution: true
max_code_length: 30000
```

## System Message Templates by Domain

### Algorithm Optimization
```
You are an expert algorithm designer specializing in {DOMAIN}.
Your task is to improve the {FUNCTION_NAME} function to achieve better {METRIC}.
The current implementation uses {CURRENT_APPROACH}.
Consider these algorithmic strategies: {STRATEGIES}.
Constraints: {CONSTRAINTS}.
The function must maintain its signature: {SIGNATURE}.
```

### Performance / Speed
```
You are a performance optimization expert.
Your task is to make {FUNCTION_NAME} run faster while maintaining correctness.
Current bottlenecks to address: {BOTTLENECKS}.
Consider: vectorization, caching, algorithmic complexity reduction, memory access patterns.
Do NOT sacrifice correctness for speed — incorrect results always score zero.
The function signature must remain: {SIGNATURE}.
```

### Data Processing Pipeline
```
You are an expert in data processing and ETL pipelines.
Your task is to optimize the {PIPELINE_NAME} for {GOAL} (throughput/memory/accuracy).
The pipeline processes {DATA_DESCRIPTION}.
Key constraints: {CONSTRAINTS}.
Focus on: efficient data structures, minimizing passes over data, reducing allocations.
```

### Machine Learning
```
You are a machine learning engineer optimizing {MODEL_TYPE}.
Your task is to improve {METRIC} (accuracy/F1/AUC/loss) on {DATASET_DESCRIPTION}.
Current approach: {CURRENT_APPROACH}.
Consider: feature engineering, hyperparameter tuning, model architecture changes.
Constraints: {CONSTRAINTS} (training time budget, inference latency, etc.)
```

### Numerical / Scientific Computing
```
You are an expert in numerical methods and scientific computing.
Your task is to improve the numerical {METHOD_TYPE} in {FUNCTION_NAME}.
Current accuracy/convergence: {CURRENT_METRIC}.
Consider: higher-order methods, adaptive step sizes, numerical stability improvements.
Maintain numerical stability — catastrophic cancellation or overflow should be avoided.
```
