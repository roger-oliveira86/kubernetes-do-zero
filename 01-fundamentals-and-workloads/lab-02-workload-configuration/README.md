# Lab 02 — Workload configuration and readiness

Use a ConfigMap to control readiness and a fictional Secret to supply an environment variable. Observe why a running Pod can become unavailable through a Service, then restore readiness without replacing the Pod.

This lab practices configuration of workloads and troubleshooting with Pod conditions, events, and EndpointSlices. It covers parts of the CKA Workloads & Scheduling and Troubleshooting domains.

## Environment

Open the [Killercoda Kubernetes playground](https://killercoda.com/playgrounds/scenario/kubernetes). Use only a disposable cluster with permission to create namespaces. The Secret contains a fictional value for this exercise. Kubernetes Secret manifests can expose their values to anyone who can read the file.

In the playground terminal, clone the repository and enter the lab directory:

```bash
git clone https://github.com/roger-oliveira86/kubernetes-from-scratch.git
cd kubernetes-from-scratch/01-fundamentals-and-workloads/lab-02-workload-configuration
test -f workload.yaml
git rev-parse HEAD
```

Record the commit used for the run.

## 1. Establish a healthy baseline

```bash
kubectl config current-context
kubectl cluster-info
kubectl version
```

Confirm that the context belongs to the playground. Then run:

```bash
kubectl create namespace lab-workload-config
kubectl -n lab-workload-config apply -f workload.yaml
kubectl -n lab-workload-config rollout status deployment/web --timeout=180s
kubectl -n lab-workload-config get pods -l app=workload-config-web -o 'custom-columns=NAME:.metadata.name,UID:.metadata.uid,RESTARTS:.status.containerStatuses[0].restartCount'
kubectl -n lab-workload-config get endpointslices -l kubernetes.io/service-name=web -o jsonpath='{.items[*].endpoints[*].conditions.ready}{"\n"}'
kubectl -n lab-workload-config exec deployment/web -- sh -c 'test -n "$DEMO_TOKEN"' && echo 'Fictional Secret is available'
```

Save the baseline name, UID, and restart count. Expect one Ready Pod, a ready endpoint, and confirmation that the environment variable exists without displaying its value.

## 2. Break readiness

```bash
kubectl -n lab-workload-config patch configmap web-readiness --type=merge -p '{"data":{"status":"blocked"}}'
kubectl -n lab-workload-config wait --for=condition=Ready=false pod -l app=workload-config-web --timeout=180s
kubectl -n lab-workload-config get pods -l app=workload-config-web
kubectl -n lab-workload-config describe pod -l app=workload-config-web
kubectl -n lab-workload-config get endpointslices -l kubernetes.io/service-name=web -o jsonpath='{.items[*].endpoints[*].conditions.ready}{"\n"}'
```

The mounted file updates asynchronously. Expect the Pod to remain Running with readiness at `0/1`, a failed probe event, and no ready Service endpoint. EndpointSlice updates can lag behind the Pod condition.

## 3. Restore readiness

```bash
kubectl -n lab-workload-config patch configmap web-readiness --type=merge -p '{"data":{"status":"ready"}}'
kubectl -n lab-workload-config wait --for=condition=Ready pod -l app=workload-config-web --timeout=180s
kubectl -n lab-workload-config get pods -l app=workload-config-web -o 'custom-columns=NAME:.metadata.name,UID:.metadata.uid,RESTARTS:.status.containerStatuses[0].restartCount'
kubectl -n lab-workload-config get endpointslices -l kubernetes.io/service-name=web -o jsonpath='{.items[*].endpoints[*].conditions.ready}{"\n"}'
```

Compare the recovered name, UID, and restart count with the baseline. All three should match, and the endpoint should be ready again. Record the Kubernetes version and the outputs for the healthy, failed, and recovered states.

## 4. Clean up

```bash
kubectl delete namespace lab-workload-config --wait=true --timeout=120s
kubectl get namespace lab-workload-config --ignore-not-found
```
