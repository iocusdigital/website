---
reviewers:
- davidopp
- mml
- foxish
- kow3ns
title: Safely Drain a Node
content_type: task
min-kubernetes-server-version: 1.5
---

<!-- overview -->
This page shows how to safely drain a {{< glossary_tooltip text="node" term_id="node" >}},
optionally respecting the PodDisruptionBudget you have defined.

## {{% heading "prerequisites" %}}

{{% version-check %}}
This task also assumes that you have met the following prerequisites:
  1. You do not require your applications to be highly available during the
     node drain, or
  1. You have read about the [PodDisruptionBudget](/docs/concepts/workloads/pods/disruptions/) concept,
     and have [configured PodDisruptionBudgets](/docs/tasks/run-application/configure-pdb/) for
     applications that need them.

<!-- steps -->

## (Optional) Configure a disruption budget {#configure-poddisruptionbudget}

To ensure that your workloads remain available during maintenance, you can
configure a [PodDisruptionBudget](/docs/concepts/workloads/pods/disruptions/).

If availability is important for any applications that run or could run on the node(s)
that you are draining, [configure a PodDisruptionBudgets](/docs/tasks/run-application/configure-pdb/)
first and then continue following this guide.

## Use `kubectl drain` to remove a node from service

You can use `kubectl drain` to safely evict all of your pods from a
node before you perform maintenance on the node (e.g. kernel upgrade,
hardware maintenance, etc.). Safe evictions allow the pod's containers
to [gracefully terminate](/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination)
and will respect the PodDisruptionBudgets you have specified.

{{< note >}}
By default `kubectl drain` ignores certain system pods on the node
that cannot be killed; see
the [kubectl drain](/docs/reference/generated/kubectl/kubectl-commands/#drain)
documentation for more details.
{{< /note >}}

When `kubectl drain` returns successfully, that indicates that all of
the pods (except the ones excluded as described in the previous paragraph)
have been safely evicted (respecting the desired graceful termination period,
and respecting the PodDisruptionBudget you have defined). It is then safe to
bring down the node by powering down its physical machine or, if running on a
cloud platform, deleting its virtual machine.

First, identify the name of the node you wish to drain. You can list all of the nodes in your cluster with

```shell
kubectl get nodes
```

Next, tell Kubernetes to drain the node:

```shell
kubectl drain <node name>
```

Once it returns (without giving an error), you can power down the node
(or equivalently, if on a cloud platform, delete the virtual machine backing the node).
If you leave the node in the cluster during the maintenance operation, you need to run

```shell
kubectl uncordon <node name>
```
afterwards to tell Kubernetes that it can resume scheduling new pods onto the node.

## Draining multiple nodes in parallel

The `kubectl drain` command should only be issued to a single node at a
time. However, you can run multiple `kubectl drain` commands for
different nodes in parallel, in different terminals or in the
background. Multiple drain commands running concurrently will still
respect the PodDisruptionBudget you specify.

For example, if you have a StatefulSet with three replicas and have
set a PodDisruptionBudget for that set specifying `minAvailable: 2`,
`kubectl drain` only evicts a pod from the StatefulSet if all three
replicas pods are [healthy](/docs/tasks/run-application/configure-pdb/#healthiness-of-a-pod);
if then you issue multiple drain commands in parallel,
Kubernetes respects the PodDisruptionBudget and ensures that
only 1 (calculated as `replicas - minAvailable`) Pod is unavailable
at any given time. Any drains that would cause the number of [healthy](/docs/tasks/run-application/configure-pdb/#healthiness-of-a-pod)
replicas to fall below the specified budget are blocked.

## The Eviction API {#eviction-api}

If you prefer not to use [kubectl drain](/docs/reference/generated/kubectl/kubectl-commands/#drain) (such as
to avoid calling to an external command, or to get finer control over the pod
eviction process), you can also programmatically cause evictions using the
eviction API.

For more information, see [API-initiated eviction](/docs/concepts/scheduling-eviction/api-eviction/).

## {{% heading "whatsnext" %}}

* Follow steps to protect your application by [configuring a Pod Disruption Budget](/docs/tasks/run-application/configure-pdb/).

