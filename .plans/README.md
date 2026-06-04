# Agent-Sandbox SWE-bench Integration - Usage Guide

This directory contains the design and implementation details for integrating GKE `agent-sandbox` with SWE-bench (via `r2egym` and `tunix`).

## Design Document
For architectural details and optimization strategies, see [agent_sandbox_swe_bench_design.md](./agent_sandbox_swe_bench_design.md).

## Quick Start / Usage

To run the evaluation with the secure `kubernetes-sandbox` backend, you need to configure the environment variables when running the `eval_deepswe.py` script.

### 1. Basic Execution (No Warmpools)
Runs tasks in sandboxed pods, provisioning them on-demand (slower startup per task).
```bash
export BACKEND="kubernetes-sandbox"
export WARMPOOL_STRATEGY="none"

python3 third_party/py/tunix/oss/examples/deepswe/eval_deepswe.py
```

### 2. Execution with Naive Parallel Warmpools
Creates warmpools for all unique images in the dataset at the start of the job.
```bash
export BACKEND="kubernetes-sandbox"
export WARMPOOL_STRATEGY="naive"
export MAX_WARMPOOL_SIZE=32 # Cap size per image warmpool

python3 third_party/py/tunix/oss/examples/deepswe/eval_deepswe.py
```

### 3. Execution with Sliding Window Warmpools (Recommended)
Sorts tasks by image and maintains a sliding window of active warmpools to optimize resource usage.
```bash
export BACKEND="kubernetes-sandbox"
export WARMPOOL_STRATEGY="sliding"
export WARMPOOL_WINDOW_SIZE=2 # Number of unique images to pre-warm concurrently
export MAX_WARMPOOL_SIZE=32   # Cap size per warmpool

python3 third_party/py/tunix/oss/examples/deepswe/eval_deepswe.py
```

## Configuration Reference

| Environment Variable | Description | Default | Allowed Values |
| :--- | :--- | :--- | :--- |
| `BACKEND` | The execution backend to use. | `kubernetes` | `kubernetes`, `docker`, `kubernetes-sandbox` |
| `WARMPOOL_STRATEGY` | Warmpool pre-warming strategy. | `none` | `none`, `naive`, `sliding` |
| `WARMPOOL_WINDOW_SIZE` | (Strategy: `sliding`) Number of unique images to pre-warm ahead in the queue. | `2` | Integer > 0 |
| `MAX_WARMPOOL_SIZE` | Maximum size (replicas) for any single image warmpool. | `32` | Integer > 0 |
| `NODE_SELECTOR_KEY` | Node selector key for sandbox templates. | `cloud.google.com/gke-nodepool` | Valid K8s label key |
| `NODE_SELECTOR_VAL` | Node selector value for sandbox templates. | `deepswe-cpu-pool` | Valid K8s label value |

## Verification & Testing
To run the basic package import verification test:
```bash
blaze test --norun_validations //third_party/py/r2egym:import_test
```
