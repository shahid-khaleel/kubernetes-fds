# k8-file-descriptors-simulator

Kubernetes manifests for a proof-of-concept that deliberately exhausts file descriptors (FDs) inside a pod, so the failure mode and its metrics can be observed with Prometheus/Grafana (see the [root README](../README.md) for the dashboard side of this).

## Contents

| File | Kind | Purpose |
|---|---|---|
| `fd-leak-pod.yaml` | `ConfigMap` + `Deployment` + `Service` | The actual FD-leak workload |
| `nginx.yaml` | `Deployment` + `Service` | Optional standalone nginx + `stress-ng` workload |

## `fd-leak-pod.yaml` — what it does mechanically

Three objects, applied together from one file:

1. **`ConfigMap/leak-sim-config`** embeds a Python script (`leak_sim.py`) as inline data:
   - Opens `/tmp/leak_fd.txt` in a loop with `os.open(..., os.O_CREAT | os.O_WRONLY)`.
   - **Never calls `os.close()`** on the returned file descriptor, and appends every FD to a Python list (`fds.append(fd)`) — this keeps a live reference so the FD can't be garbage-collected or reused.
   - Starts at `leak_rate = 1` FD/sec and increments the rate every ~60 FDs opened, so the leak accelerates over time instead of running at a constant (and possibly too-slow-to-notice) pace.
   - Registers a `SIGTERM` handler that logs how many FDs were open at the moment of shutdown, then exits — so a `kubectl delete`/pod eviction produces a clean, informative log line instead of a silent kill.
   - Because the same file path is reused (`/tmp/leak_fd.txt`) and never closed, the container will eventually hit the process's open-file-descriptor limit (`ulimit -n`, default ~1024 in most container images) and Python will raise `OSError: [Errno 24] Too many open files`.

2. **`Deployment/fd-leak-sim`** runs one replica of a `python:3.9-slim` container named `leaker`:
   - Mounts the ConfigMap as `/app/leak_sim.py` (`defaultMode: 0755`, executable).
   - Startup command runs `apt-get update && apt-get install -y procps` before launching the script — `procps` is not required by the script itself but is useful for `ps`/process inspection from inside the container while debugging.
   - Sets CPU/memory `requests`/`limits` (`50m`/`64Mi` request, `100m`/`128Mi` limit) so the demo isolates FD exhaustion rather than also causing an OOM kill or CPU throttling event.
   - `terminationGracePeriodSeconds: 30` gives the `SIGTERM` handler time to log before force-kill.
   - **Pins `spec.nodeName: test`.** This is a hardcoded scheduling constraint left over from development/testing — unless your cluster has a node literally named `test`, the pod will sit in `Pending`. Remove this field (or set it to a real node name from `kubectl get nodes`) before applying.

3. **`Service/fd-leak-sim-svc`** is a `ClusterIP` service on port 80. The comment in the file (`# Not really used, but placeholder`) confirms this is not functionally required — the leak script has no HTTP server — it exists so the Deployment has an associated Service, e.g. for teams that want a consistent object set for testing service-discovery/DNS behavior alongside the leak, or simply as a template placeholder.

## `nginx.yaml` — how it relates

This manifest deploys a **separate**, unrelated `Deployment/nginx-stress` (nginx + `stress-ng`, CPU/memory limits `500m`/`512Mi`) and its `Service/nginx-stress-svc`. Nothing in either manifest wires the two workloads together (no shared `nodeName`, no anti-affinity, no scheduling hint) — they are independent objects that happen to live in the same folder.

The most defensible read of its purpose, based on the leak script's own header comments ("Testing multiple simultaneous leakers", "Observing impact on other pods"): it is meant to be **manually co-located** with `fd-leak-sim` (e.g. by pinning both to the same node) to act as a "noisy neighbor" — a normal, healthy workload you can watch for side effects (latency, restarts, OOM) once the leaking pod starts exhausting node-level file descriptors. It is not required to reproduce the core FD-leak behavior and can be skipped if you only want to observe the leak in isolation.

## How to Run / Observe

```bash
# Deploy the FD-leak workload (edit/remove nodeName first — see above)
kubectl apply -f fd-leak-pod.yaml

# Watch scheduling and restarts
kubectl get pods -w

# Tail the leak ramp
kubectl logs -f deployment/fd-leak-sim

# Confirm FD count from inside the container
kubectl exec -it <pod-name> -- ls /proc/self/fd | wc -l

# Optional: deploy the noisy-neighbor workload too
kubectl apply -f nginx.yaml

# Clean up
kubectl delete -f fd-leak-pod.yaml
kubectl delete -f nginx.yaml
```

Expected sequence: FD count climbs steadily in the logs → leak rate increases roughly every 60 FDs → the process eventually throws `OSError: [Errno 24] Too many open files` → the container exits → Kubernetes restarts it per the Deployment's default restart policy → the cycle repeats. If Prometheus/Grafana are wired up against the cluster, this shows up as a rising `container_file_descriptors{container="leaker"}` line and, if the leak is severe/long-running enough, movement in node-level `node_filefd_allocated` — see the panel breakdown in the [root README](../README.md#grafana-dashboard).

## Notes / Caveats

- Both manifests run `apt-get update && apt-get install` at container startup, which requires outbound network access from the pod and adds startup latency — expected for a quick lab demo, not a pattern for production images.
- Neither container sets a `securityContext` (no `runAsNonRoot`, no capability drops). Fine for a disposable test namespace; harden before reusing the pattern elsewhere.
- Only apply these manifests in a non-production, disposable namespace/cluster — see the [Disclaimer in the root README](../README.md#disclaimer).
