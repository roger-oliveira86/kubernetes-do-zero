# Lab 01 — First commands with kubectl

In this lab, you'll check the cluster, create a Pod, and use the first Kubernetes diagnostic commands.

## Objectives

By the end, you'll know how to:

- check the cluster's nodes and namespaces;
- create a Pod from a YAML manifest;
- check the Pod's state;
- inspect events and details;
- check logs;
- remove the created resource.

## Prerequisites

- access to a disposable Kubernetes cluster;
- `kubectl` installed and connected to the cluster;
- a terminal open in this folder.

You can also use a browser-based lab environment, such as Killercoda.

## 1. Check the connection to the cluster

```bash
kubectl config current-context
LAB_CONTEXT="$(kubectl config current-context)"
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" cluster-info
```

Confirm that the context and API server belong to the disposable cluster you opened for this exercise. Stop if either is unexpected. Run the commands in the same shell.

The result should show the addresses of the control plane and the main services.

## 2. List the nodes

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" get nodes
```

Expected result: at least one node in the `Ready` state.

## 3. List the namespaces

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" get namespaces
```

You should find namespaces like `default`, `kube-system` and `kube-public`.

## 4. Create the first Pod

Create a fresh namespace and stop if creation fails, including if it already exists.

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" create namespace lab-first-commands
```

After namespace creation succeeds, apply the manifest:

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-first-commands apply -f pod.yaml
```

Expected result:

```text
pod/first-pod created
```

## 5. Track the Pod's state

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-first-commands get pods
```

If the image is still being pulled, the state may temporarily show as `ContainerCreating`. Wait a few seconds and run the command again. The expected final state is `Running`.

To follow the change in real time, use:

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-first-commands get pods --watch
```

Press `Ctrl+C` to stop watching.

## 6. Inspect the Pod

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-first-commands describe pod first-pod
```

Pay special attention to:

- `Status`;
- `Containers`;
- `Conditions`;
- `Events`, at the end of the output.

The events help you understand scheduling problems, image pulls and container startup.

## 7. Check the logs

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-first-commands logs first-pod
```

Expected result:

```text
The first Pod is working.
```

To follow new messages:

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-first-commands logs -f first-pod
```

Press `Ctrl+C` to exit.

## 8. Query using labels

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-first-commands get pods -l app=first-pod
```

Kubernetes will return only the Pods that have the label `app=first-pod`.

## 9. Clean up the environment

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-first-commands delete -f pod.yaml
```

Expected result:

```text
pod "first-pod" deleted
```

Confirm the removal:

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-first-commands get pods
```

## Optional challenge

Run this before the final namespace cleanup.

Change the message in the `pod.yaml` file, apply the manifest again, and check the new logs. Since the Pod's `command` field can't be changed directly, remove the Pod before recreating it:

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-first-commands delete -f pod.yaml
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-first-commands apply -f pod.yaml
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" -n lab-first-commands logs first-pod
```

## Final namespace cleanup

Confirm the recorded context still points to your disposable cluster and that you created the namespace in this run. Delete the namespace after the exercise and any optional challenge. The final command should produce no output.

```bash
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" delete namespace lab-first-commands
kubectl --context="${LAB_CONTEXT:?Verify and set LAB_CONTEXT first}" get namespace lab-first-commands --ignore-not-found
```

## Checklist

- [ ] The cluster responded to `kubectl cluster-info`.
- [ ] At least one node showed up as `Ready`.
- [ ] The Pod reached the `Running` state.
- [ ] The `describe` command showed details and events.
- [ ] The `logs` command displayed the application's message.
- [ ] The Pod was removed at the end.
