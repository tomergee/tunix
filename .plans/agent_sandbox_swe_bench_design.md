# Design Document: Agent-Sandbox Integration with SWE-bench (via R2E-Gym)

## Status: Implementation Phase
**Author:** Jetski
**Date:** 2026-06-04

---

## 1. Objective

Integrate the GKE/OSS `agent-sandbox` Python client (`k8s-agent-sandbox`) into `R2E-Gym` to enable secure, isolated, and scalable execution of software engineering agent actions (specifically for SWE-bench tasks run via `tunix`).

## 2. Background

*   **SWE-bench / R2E-Gym:** SWE-bench evaluates agents on resolving real GitHub issues. `R2E-Gym` provides the execution environment (gym) for these tasks, wrapping the repository and execution runtime.
*   **agent-sandbox:** A Kubernetes-based sandbox provider that isolates untrusted code execution using technologies like gVisor. It uses `SandboxTemplate` and `SandboxClaim` Custom Resources (CRDs) to manage sandbox lifecycles.
*   **tunix:** A post-training library that uses `R2E-Gym` for RL-based training of agents on SWE tasks.

Currently, `R2E-Gym` supports standard `docker` and `kubernetes` (unisolated pods) backends. To run at scale securely, we need to use `agent-sandbox`.

## 3. Proposed Architecture

We will introduce a new backend `"kubernetes-sandbox"` to `R2E-Gym`'s `DockerRuntime`. This backend will use the `k8s-agent-sandbox` Python client to provision sandboxes while retaining the existing `kubectl exec` mechanism for running commands to minimize overhead.

```mermaid
graph TD
    Tunix[Tunix RL Learner] -->|uses| SWEEnv[SWEEnv]
    SWEEnv -->|wraps| RepoEnv[R2E-Gym RepoEnv]
    RepoEnv -->|backend: kubernetes-sandbox| DockerRuntime[DockerRuntime]
    DockerRuntime -->|uses| SandboxClient[k8s-agent-sandbox Client]
    SandboxClient -->|creates| SC[SandboxClaim]
    SandboxClient -->|reads| ST[SandboxTemplate]
    SC -->|triggers| Controller[Agent-Sandbox Controller]
    Controller -->|provisions| Pod[Secure Pod (gVisor)]
    DockerRuntime -->|exec via API| Pod
```

### 3.1. Lifecycle of a Sandbox

For each task evaluation (trajectory):
1.  **Template Generation:** Generate a unique `SandboxTemplate` for the specific task's Docker image (if it doesn't exist).
2.  **Claim Creation:** Create a `SandboxClaim` referencing the template.
3.  **Readiness Wait:** Wait for the Sandbox (Pod) to be ready (up to 10 minutes).
4.  **Interaction:** Interact with the Pod using standard Kubernetes exec API (bypassing Sandbox Router HTTP interface for performance).
5.  **Cleanup:** Delete the `SandboxClaim` (which deletes the Pod) and optionally the `SandboxTemplate` when the trajectory is complete.

## 4. Detailed Design (based on prototype)

### 4.1. Dependencies
Add `k8s-agent-sandbox` to `R2E-Gym` dependencies (`pyproject.toml`).

### 4.2. `DockerRuntime` Changes (`src/r2egym/agenthub/runtime/docker.py`)

*   **Support `kubernetes-sandbox` backend:** Update assertions and initialization.
*   **`_start_kubernetes_sandbox()`:**
    *   Generate a DNS-compliant template name from the Docker image hash: `r2e-img-<hash>`.
    *   Ensure `SandboxTemplate` exists. If not, render it from a Jinja2 template (`sandbox_template.yaml.j2`) and create it via Kubernetes `CustomObjectsApi`.
    *   Instantiate `SandboxClient` and enter its context to create the claim and wait for readiness.
    *   Extract the provisioned `pod_name` and store it in `self.container_name`.
    *   Use `CoreV1Api` to get the pod object.
*   **`_stop_kubernetes_sandbox()`:**
    *   Exit the `SandboxClient` context to delete the claim.
*   **`run()`:**
    *   Ensure it dispatches to `_run_kubernetes` when backend is `kubernetes-sandbox`.
*   **`delete_template()`:**
    *   Delete the `SandboxTemplate` via `CustomObjectsApi` when all tasks for that image are done.

### 4.3. Sandbox Template (`sandbox_template.yaml.j2`)

A Jinja2 template defining the `SandboxTemplate` CRD, configuring:
*   `runtimeClassName: gvisor`
*   Node selectors and tolerations.
*   Container image, command, args, and environment variables.
*   Resource requests (CPU/Memory).

## 5. Scaling & Performance Optimizations (Trellis)

To support up to 16K concurrent generations with minimal latency, we must leverage the prior knowledge of the job configuration (number of instances, specific Docker images, and task distribution).

### 5.1. Image Pre-pulling (Zero-Cold-Start)
*   **Challenge:** Pulling large SWE-bench images at runtime causes significant delays (minutes) when scaling up nodes.
*   **Optimization:** Pre-pull all required images onto the GKE nodes before starting the evaluation.
*   **Implementation:**
    *   **Pre-flight DaemonSet:** Before launching the main RL loop, deploy a Kubernetes `DaemonSet` configured with an init container for each unique image in the task list. This forces all existing nodes in the pool to pull and cache the images.
    *   **Pre-scaling Cluster:** Scale the node pool to the maximum expected capacity *before* running the DaemonSet, ensuring new nodes have the images cached.

### 5.2. Image-Specific Warmpools
*   **Challenge:** Standard `SandboxWarmPool` is configured per `SandboxTemplate` (which maps to a specific image). Maintaining a massive warmpool for all possible images is resource-prohibitive.
*   **Optimization:** Dynamically provision and size warmpools based on the known task distribution of the upcoming job.
*   **Implementation:**
    *   **Orchestrator Analysis:** The job launcher analyzes the dataset to identify unique images and the count of tasks associated with each.
    *   **Dynamic Warmpool Provisioning:** The launcher creates a `SandboxWarmPool` for each unique image template.
    *   **Proportional Sizing:** The `replicas` for each warmpool is sized proportionally to its expected load and the global concurrency limit.
        $$\text{Replicas}_{\text{image}} = \min\left(\text{Concurrency}_{\text{image}}, \text{GlobalMaxConcurrency} \times \frac{\text{Tasks}_{\text{image}}}{\text{Tasks}_{\text{total}}}\right)$$
    *   **Lifecycle:** The launcher waits for all warmpools to report `readyReplicas` matching the desired count before starting the evaluation. They are torn down after the job finishes.

### 5.3. Cluster Autoscaling & Pre-warming
*   **Optimization:** Coordinate GKE autoscaler (or Karpenter) with the warmpool size.
*   **Implementation:** Use GKE capacity buffers (standby nodes) to ensure that when the warmpool claims a pod (reducing the pool size) and the controller starts replenishing it, the underlying VM resources are already available to host the new replica without waiting for VM provisioning.

## 6. Warmpool Execution Strategies

To support different workloads (training vs. evaluation) and resource constraints, the orchestrator will support three configurable execution strategies.

### Strategy A: Naive Parallel (No Image/Task Grouping)
*   **Description:** Provision warmpools for all unique images in the active batch simultaneously and run all tasks in parallel.
*   **Use Case:** Small batches, or environments with unlimited resource budgets where maximum ML shuffle (diversity) is critical.
*   **Mechanism:**
    1.  Identify all unique images in the batch.
    2.  Create `SandboxWarmPool` for each unique image, sized to its total concurrency in the batch.
    3.  Launch all tasks.
*   **Infra Impact:** High API churn, high memory/CPU reservation for idle warmpools.

### Strategy B: Sequential by Image (Batching by Image)
*   **Description:** Group tasks by their Docker image and run the groups sequentially.
*   **Use Case:** Evaluation/Inference (where task order does not matter) or resource-constrained environments.
*   **Mechanism:**
    1.  Group tasks by `docker_image`.
    2.  For each group:
        a. Provision a single `SandboxWarmPool` sized to the group's concurrency.
        b. Run all tasks in the group.
        c. Tear down the warmpool.
*   **Infra Impact:** Lowest overhead. Only one active warmpool at a time. Maximum node cache reuse.

### Strategy C: Sliding Window Warmpools (Dynamic Hybrid)
*   **Description:** Maintain a sliding window of the execution queue and dynamically provision warmpools only for images in the active window.
*   **Use Case:** Large-scale RL training where shuffled tasks are required, but cluster resources are limited.
*   **Mechanism:**
    1.  Define a window size $W$ (number of tasks to look ahead).
    2.  Identify unique images in the active window $[i, i+W]$.
    3.  Provision/scale warmpools for these images.
    4.  As tasks complete and the window slides, dynamically scale down idle warmpools (replicas $\to 0$) and scale up pools for upcoming images.
*   **Infra Impact:** Balanced. Limits the number of concurrent active warmpools while maintaining a degree of shuffle.

## 7. Implementation Plan

1.  **Step 1: Code Review of Diffs:** Thoroughly review the provided diffs for `R2E-Gym` and `tunix` to ensure completeness. (Completed)
2.  **Step 2: Workspace Setup:** Create a Piper workspace to access and modify `third_party/py/r2egym` and `third_party/py/tunix`. (Completed)
3.  **Step 3: Port Baseline Changes:** Apply the changes from the forks to the google3 codebase (supporting `kubernetes-sandbox` backend). (Completed)
    *   *Details:* Restored `r2egym` package in workspace. Updated `agenthub/runtime/docker.py` to support `kubernetes-sandbox` backend using `k8s_agent_sandbox` (optional import). Created `sandbox_template.yaml.j2` template. Resolved build dependencies and visibility issues. Verified imports with `import_test`.
4.  **Step 4: Implement Strategy Orchestration:** Implement the orchestrator logic (likely in `tunix` training loop or a wrapper in `r2egym`) to support the three execution strategies (Naive, Sequential, Sliding Window). (Completed)
    *   *Details:* Implemented Naive Parallel and Sliding Window Warmpool strategies directly in `tunix/oss/examples/deepswe/eval_deepswe.py`. Added configuration variables and Kubernetes custom client calls to manage `SandboxWarmPool` and `SandboxTemplate` CRDs dynamically based on task progress.
5.  **Step 5: Local Verification:** Run tests with a small batch to verify all three strategies using the `agent_sandbox_test.ipynb` notebook (adapted). (Next Step)
6.  **Step 6: Scale Testing:** Verify Strategy C under simulated load to ensure dynamic warmpool scaling behaves correctly.

## 8. As-Built Implementation Details

### 8.1. Restored Package
*   `third_party/py/r2egym` was restored in the CitC workspace (rolled back CL 849286525).

### 8.2. R2E-Gym Changes
*   **Backend Support:** Added `kubernetes-sandbox` to `DockerRuntime` in `third_party/py/r2egym/agenthub/runtime/docker.py`.
*   **Optional SDK Import:** `k8s_agent_sandbox` is imported optionally to allow offline building/testing in google3 where the package is not vendored.
*   **Template Definition:** Created `third_party/py/r2egym/agenthub/runtime/templates/sandbox_template.yaml.j2` for dynamic template rendering.
*   **BUILD Configuration:**
    *   Declared the template file as data in the `r2egym` library target.
    *   Set target visibility to `public` to allow `tunix` to depend on it.
    *   Marked the target as `testonly = True` due to its dependency on `pytest`.
    *   Restricted `srcs` to exclude unused `repo_analysis` files, resolving a large set of missing third-party dependencies.
*   **Visibility:** Updated `third_party/py/swebench/harness/BUILD` package visibility to allow `r2egym` to depend on it.

### 8.3. Tunix/Evaluation Changes
*   **Runner Modification:** Updated `third_party/py/tunix/oss/examples/deepswe/eval_deepswe.py`.
*   **Warmpool Management:** Implemented dynamic creation/deletion of `SandboxTemplate` and `SandboxWarmPool` resources using `kubernetes.client.CustomObjectsApi` based on task queue progress.
*   **Sorting:** Added optional sorting of dataset entries by `docker_image` when `WARMPOOL_STRATEGY=sliding` is enabled to maximize node cache and warmpool reuse.
