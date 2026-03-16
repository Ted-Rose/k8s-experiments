# Quota Rollout Failure — Local Test

Demonstrates how a pod memory-request increase can stall a rolling deployment
when the namespace `ResourceQuota` leaves insufficient headroom for the surge pod.

## Cluster setup (GKE)

```bash
gcloud container clusters create quota-test-cluster \
    --num-nodes=2 \
    --machine-type=e2-medium \
    --zone=us-central1-a

gcloud container clusters get-credentials quota-test-cluster --zone=us-central1-a
```

## Apply baseline state

```bash
kubectl apply -f setup.yaml
```

**Quota:** 200Mi
**Initial usage:** 3 pods × 50Mi = 150Mi  (50Mi headroom)

## Trigger the failure

```bash
kubectl apply -f deploy-breaking.yaml
```

Kubernetes attempts a rolling update with `maxSurge=25%` (1 extra pod).
Before any old pod is terminated it tries to schedule the surge pod:

```
150Mi (existing) + 60Mi (surge) = 210Mi  >  200Mi quota  →  FORBIDDEN
```

## Verify the stalled rollout

```bash
# Rollout hangs at 3/3 desired, 0 updated
kubectl get deployments -n quota-demo

# ReplicaSet shows the quota error
kubectl describe rs -n quota-demo | grep -A2 -i "failed\|forbidden"
```

Expected event:

```
FailedCreate  pods "app-v1-xxxx" is forbidden: exceeded quota: mem-quota,
              requested: requests.memory=60Mi, used: requests.memory=150Mi,
              limited: requests.memory=200Mi
```
