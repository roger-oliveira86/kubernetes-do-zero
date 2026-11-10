# Lab 01 — Deployment, Service, Rollout and Rollback

This lab turns an isolated Pod into a declared, replicated, updatable application. The goal isn't just to run commands: it's to observe how a Deployment reduces risk by keeping replicas available during a change, and how a rollback returns the system to a known revision.

## What you'll learn

By the end, you'll be able to:

- create an isolated namespace for the lab;
- apply a Deployment with three replicas and a RollingUpdate strategy;
- expose the Pods through a ClusterIP Service;
- follow an update triggered by a template change;
- recognize a rollout that isn't progressing;
- use `kubectl rollout undo` to return to the previous revision;
- clean up every resource created.

## Prerequisites

- an accessible Kubernetes cluster;
- `kubectl` configured for the lab context;
- permission to create a namespace, Deployment and Service;
- a terminal open in this folder.

## Where to run it

Pick one disposable environment. Before provisioning locally, inspect your current context with `kubectl config current-context` and list the existing clusters or profiles using the command below. A missing current context is expected on a fresh setup.

| Option | Check before creation | Create and verify |
| --- | --- | --- |
| [Killercoda Kubernetes Playground](https://killercoda.com/playgrounds/scenario/kubernetes) | Open a fresh playground session. | Use the context provided by that session. |
| [kind](https://kind.sigs.k8s.io/) | Run `kind get clusters`. Stop if `kubernetes-from-scratch` already exists. | Run `kind create cluster --name kubernetes-from-scratch`. Expect context `kind-kubernetes-from-scratch`. |
| [minikube](https://minikube.sigs.k8s.io/docs/start/) | Run `minikube profile list`. Stop if `kfs-lab-rollout` already exists, even if stopped. | Run `minikube start -p kfs-lab-rollout`, then `minikube status -p kfs-lab-rollout`. Expect context `kfs-lab-rollout`. |

If you cannot check the inventory, stop. Use a fresh environment and leave existing clusters and profiles alone. After creation, check the context, server and version:

```bash
kubectl config current-context
LAB_CONTEXT="$(kubectl config current-context)"
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" cluster-info
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" version
```

Confirm the context and API server match the environment you just created. Stop if either is unexpected. Run the remaining commands in this shell. If you open another terminal, repeat context verification there before using it for lab commands.

## Files

| File | Responsibility |
| --- | --- |
| `namespace.yaml` | isolates the lab's resources |
| `deployment.yaml` | declares three replicas, the update strategy and a readiness probe |
| `service.yaml` | provides a stable address for the selected Pods |

The Deployment pins NGINX to a multi-architecture OCI digest so the lab pulls the same image for a given platform even if the tag moves. The tag identifies the NGINX release.

## 1. Create the initial version

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" create -f namespace.yaml
```

Continue only after namespace creation succeeds. If it already exists, stop and use a fresh environment.

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-rollout apply -f deployment.yaml
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-rollout apply -f service.yaml

kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-rollout rollout status deployment/web
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-rollout get deployment,rs,pods,service
```

Expected result: the Deployment reports `successfully rolled out`, three Pods are ready, and the `web` Service has endpoints.

Check the endpoints:

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-rollout get endpointslices -l kubernetes.io/service-name=web
```

Optionally, in another terminal, forward a local port to the Service:

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-rollout port-forward service/web 8080:80
```

Then access `http://localhost:8080` in your browser or with:

```bash
curl -I http://localhost:8080
```

## 2. Observe a controlled rollout

Change an environment variable in the Pod template. The NGINX application doesn't use this variable; it exists to demonstrate a declarative change that creates a new revision without depending on a specific image tag.

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-rollout set env deployment/web RELEASE_MESSAGE=v2
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-rollout rollout status deployment/web
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-rollout rollout history deployment/web
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-rollout get rs
```

Notice that the Deployment creates a new ReplicaSet and gradually scales down the previous one. The `maxUnavailable: 1` and `maxSurge: 1` limits make the trade-off explicit: during the update, at most one desired replica is unavailable and at most one extra replica is created.

## 3. Simulate a change that never becomes ready

Now deliberately introduce a nonexistent image. This is a controlled exercise; don't use this pattern to test production.

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-rollout set image deployment/web web=nginx:nonexistent-image
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-rollout rollout status deployment/web --timeout=60s
```

The command should end with a timeout, since the new Pod won't be able to pull the image. Investigate before fixing it:

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-rollout get pods
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-rollout describe deployment web
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-rollout describe pod -l app.kubernetes.io/name=web
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-rollout get events --sort-by=.metadata.creationTimestamp
```

Look for messages like `ErrImagePull` or `ImagePullBackOff`. The decision isn't "retry until it works" — it's comparing the current state against the impact and returning to a known revision when the new one isn't progressing.

## 4. Run the rollback

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-rollout rollout undo deployment/web
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-rollout rollout status deployment/web
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-rollout rollout history deployment/web
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-rollout get pods
```

Expected result: the Deployment returns to the last healthy revision and is back to three ready Pods.

## 5. Clean up the environment

Confirm that the recorded context still points to your disposable cluster and that you created the namespace in this run.

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" delete namespace lab-rollout
```

Confirm:

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" get namespace lab-rollout
```

The namespace should stop appearing once deletion completes.

Before deleting a local cluster, check the recorded context and list the clusters or profiles again with `kind get clusters` or `minikube profile list`. Confirm the name still identifies the environment you created for this run. If there is any doubt, leave it in place.

- For kind, run `kind delete cluster --name kubernetes-from-scratch`.
- For Minikube, run `minikube delete -p kfs-lab-rollout`.
- For Killercoda, end the playground session.

## Review questions

1. Why does changing an environment variable create a new Deployment revision?
2. What's the difference between an individual Pod's state and the Deployment's desired state?
3. What evidence showed that the invalid image wasn't a slow rollout, but an update that couldn't complete?
4. In what situation would you choose to pause an update instead of running a rollback?

## References

- [Deployments — Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Service — Kubernetes](https://kubernetes.io/docs/concepts/services-networking/service/)
- [kubectl rollout undo — Kubernetes](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_undo/)
