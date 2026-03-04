---

# FD Leak Simulation in Kubernetes

This project simulates a file descriptor (FD) leak inside a Kubernetes pod to test observability, alerting, and system behavior under resource exhaustion.

The goal is to intentionally exhaust file descriptors in a controlled environment and observe how Kubernetes, the container runtime, and monitoring systems respond.

---

## Overview

The deployment includes:

* A **ConfigMap** containing a Python script that leaks file descriptors
* A **Deployment** running the script in a container
* A **ClusterIP Service** (placeholder)

The Python script:

* Opens file descriptors using `os.open()`
* Does not close them
* Stores references to prevent garbage collection
* Gradually increases the leak rate
* Handles `SIGTERM` for graceful shutdown logging

Over time, the container reaches the operating system file descriptor limit (typically ~1024 by default), resulting in:

```
OSError: [Errno 24] Too many open files
```

This allows testing of failure scenarios and observability configurations.

---

## Why This Project Exists

File descriptor leaks are a common production issue caused by:

* Unclosed HTTP connections
* Database connection leaks
* Improper file handling
* Logging misconfiguration
* gRPC stream leaks

Unlike CPU or memory spikes, FD exhaustion often degrades systems silently before failure.

This project is designed to:

* Test Kubernetes restart behavior
* Observe node vs container-level impact
* Validate Prometheus metrics
* Configure Grafana dashboards
* Test alerting rules
* Practice incident analysis

---

## Architecture

```
ConfigMap → Python FD Leak Script
        ↓
Deployment (1 replica)
        ↓
Container gradually consumes file descriptors
        ↓
Prometheus collects metrics
        ↓
Grafana visualizes behavior
```

---

## Deployment

Apply the manifests:

```bash
kubectl apply -f fd-leak.yaml
```

Check pod status:

```bash
kubectl get pods
```

Stream logs:

```bash
kubectl logs -f deployment/fd-leak-sim
```

Inspect open file descriptors inside the container:

```bash
kubectl exec -it <pod-name> -- ls /proc/self/fd | wc -l
```

---

## Expected Behavior

1. File descriptors increase steadily.
2. Leak rate increases over time.
3. Eventually the process hits the FD limit.
4. Python throws `Too many open files`.
5. The container may exit.
6. Kubernetes restarts the pod (depending on failure behavior).
7. Metrics reflect the exhaustion event.

---

## Metrics to Monitor (Prometheus)

### Node-Level Metrics

```
node_filefd_allocated
node_filefd_maximum
```

These show total allocated file descriptors versus system maximum.

---

### Container-Level Metrics

```
container_file_descriptors
container_memory_usage_bytes
container_cpu_usage_seconds_total
```

The primary metric of interest is `container_file_descriptors`.

---

### Restart Tracking

```
kube_pod_container_status_restarts_total
```

Used to verify Kubernetes restart behavior.

---

## Grafana Dashboard Configuration

### Panel 1 – Node FD Usage

PromQL:

```
node_filefd_allocated
node_filefd_maximum
```

Visualization:

* Time series
* Add threshold at 80%
* Compare allocated vs maximum

---

### Panel 2 – Container File Descriptors

PromQL:

```
container_file_descriptors{container="leaker"}
```

This should show a steadily increasing line.

---

### Panel 3 – Restart Count

```
kube_pod_container_status_restarts_total{container="leaker"}
```

Tracks restart activity after exhaustion.

---

### Panel 4 – Memory Usage

```
container_memory_usage_bytes{container="leaker"}
```

Confirms the issue is FD exhaustion rather than memory exhaustion.

---

## Example Alert Rules

### Alert: Node FD Usage > 80%

```
node_filefd_allocated / node_filefd_maximum > 0.8
```

---

### Alert: Rapid FD Growth

```
rate(container_file_descriptors[5m]) > 5
```

Detects accelerated FD consumption.

---

## Optional Experiments

You can extend this test by:

* Lowering `ulimit` inside the container
* Increasing replica count
* Removing fixed `nodeName`
* Testing multiple simultaneous leakers
* Observing impact on other pods
* Comparing behavior with memory leaks

---

## Resource Configuration

Container limits:

```yaml
resources:
  limits:
    cpu: "100m"
    memory: "128Mi"
  requests:
    cpu: "50m"
    memory: "64Mi"
```

These limits ensure the experiment isolates FD exhaustion rather than CPU or memory exhaustion.

---

## Cleanup

Remove all resources:

```bash
kubectl delete -f fd-leak.yaml
```

---

## Key Learning Outcomes

* Understand Linux file descriptor limits
* Observe Kubernetes failure handling
* Validate monitoring coverage
* Test alert reliability
* Improve production readiness

---

## Disclaimer

This project is intended for testing in non-production environments only.

Do not deploy in production clusters without understanding the potential node-level impact.

---

## Future Extensions

* Memory leak simulation
* Ephemeral storage exhaustion
* CPU starvation scenarios
* Network socket exhaustion
* Chaos engineering automation

---

By intentionally exhausting system resources in a controlled environment, teams can improve observability maturity and resilience before real incidents occur.

