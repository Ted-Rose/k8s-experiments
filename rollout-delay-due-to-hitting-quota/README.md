# Rollout Delay Due to Hitting Quota

Demonstrates how a pod memory-request increase can stall a rolling deployment
when the namespace `ResourceQuota` leaves insufficient headroom for the surge pods.

## Experiment

### Step 0: Clean up previous experiment cluster

```bash
kind delete cluster --name quota-test
```

### Step 1: Set up test cluster

```bash
brew install kind
kind create cluster --name quota-test
# kubectl context is set automatically to kind-quota-test
```

> **Note (kind only):** `nginx:alpine` must be pre-loaded because kind nodes have no
> direct internet access by default.

```bash
docker pull nginx:alpine
kind load docker-image nginx:alpine --name quota-test
```

### Step 2: Apply baseline state

```bash
kubectl apply -f setup.yaml
```

### Step 3: Check namespace and quota

```bash
kubectl get resourcequota -n quota-demo
```

Should output:

```bash
NAME        REQUEST                        LIMIT   AGE
mem-quota   requests.memory: 400Mi/500Mi           9s
```

### Step 4: Watch events

```bash
kubectl get events -n quota-demo --watch
```

### Step 5: Trigger the rollout delay

```bash
kubectl apply -f deploy-quota-exceed.yaml
```

Kubernetes attempts a rolling update with the default `maxSurge=25%` (2 extra pods for 8 replicas).
Before any old pod is terminated it tries to schedule both surge pods:

```text
400Mi (existing 8 pods) + 2 × 52Mi (surge pods) = 504Mi  >  500Mi quota  →  FORBIDDEN
```

One surge pod succeeds (452Mi used), but the second is rejected. The rollout proceeds
one pod at a time, and each rejected attempt triggers the ReplicaSet controller's
exponential backoff (up to ~16 minutes), making the rollout extremely slow.

## Verify the delayed rollout

```bash
# Rollout progresses very slowly — check updated vs desired replicas
kubectl get deployments -n quota-demo

# ReplicaSet shows the quota error
kubectl describe rs -n quota-demo | grep -A2 -i "failed\|forbidden"
```

## Findings

The rollout did not fail — it completed, but took over 3 hours instead of seconds.

The root cause is the ReplicaSet controller's exponential backoff. When a surge pod
is rejected by the quota (`FORBIDDEN`), the controller backs off before retrying.
The default backoff parameters (defined in [client-go/util/workqueue/default_rate_limiters.go](https://github.com/kubernetes/client-go/blob/master/util/workqueue/default_rate_limiters.go)) are:

| Parameter | Value |
|---|---|
| Base delay | 5ms |
| Max delay | 1000s (~16 min) |
| Factor | 2 (doubles each retry) |

After a few `FailedCreate` errors the retry interval hits the 16-minute cap. Even
though quota frees up as soon as an old pod terminates, the controller is still
waiting out its backoff timer and does not retry immediately. This turns a
one-pod-at-a-time rollout into a multi-hour crawl.

This is a known issue: [kubernetes/kubernetes#98656](https://github.com/kubernetes/kubernetes/issues/98656).
