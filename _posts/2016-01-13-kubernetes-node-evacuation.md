---
layout: post
title: Kubernetes node evacuation
date:   2016-01-13 08:21:32
categories: kubernetes api kubectl
---

"Kubernetes is great in increasing the cluster size by adding new nodes. But it has no way to evacuate and cleanly remove a node." I have heard this multiple time. Is this true?

In the following I will show how it is easily possible with a bit of bash scripting to evacuate a node. Kubernetes is a very flexible and scriptable system. At the heart of nearly all logic provided by the system itself are nothing more than calls to the apiserver. If there is a feature that seems to be missing, often it is easy to add it with a few API or kubectl calls.

We want to evacuate a node. Evacuation means that all pods must be terminated cleanly, preferably following defined termination grace periods.

## Step 1 – unschedulable

Deleting pods is easy. But if the scheduler is fast enough (and it probably is if some replication controller wants to spawn new pods), it will quickly launch new pods on the evacuated node. Hence, we have to disable the node for the scheduler first. This is a one-liner:

```bash
$ kubectl patch node gke-cluster-1-b4c97d4d-node-psh2 \
    -p '{"spec":{"unschedulable":true}}'
```

Setting the `unschedulable` flag makes the scheduler to ignore that node. All pods keep running.

## Step 2 - get pods

Which pods are running on a node? Simple question, but trickier than expected. Unfortunately, kubectl cannot filter by node ([Github issue 14129](https://github.com/kubernetes/kubernetes/issues/14129)).

The immediate solution would be to list all pods and use some bash magic to filter by node. But, we can do better:

Kubectl has a JSONPath output format (compare the [docs](https://github.com/kubernetes/kubernetes/blob/release-1.1/docs/user-guide/jsonpath.md)). JSONPath supports filtering using the `?()` filter syntax. With this we are all set:

```bash
$ kubectl get pods --all-namespaces \
    -o jsonpath='{range .items[?(.spec.nodeName=="gke-cluster-1-b4c97d4d-node-psh2")]}{.metadata.name} {end}'
busybox-694uu fluentd-cloud-logging-gke-cluster-1-b4c97d4d-node-psh2
```

This gives us a space separated list of pod names.

Because we use `--all-namespaces`, we also see other namespaces than the `default` one, e.g. the kube-system namespace with the logging daemon fluentd (on GKE). If we try to delete those pod names, the fluentd deletion will fail. We have to add namespaces to the output. That's easy:

```bash
$ kubectl get pods --all-namespaces \
    -o jsonpath='{range .items[?(.spec.nodeName=="gke-cluster-1-b4c97d4d-node-psh2")]}{.metadata.namespace} {.metadata.name} {end}'
default busybox-694uu kube-system fluentd-cloud-logging-gke-cluster-1-b4c97d4d-node-psh2
```

This looks much better already.

## Step 3 - deleting the pods

We have a list of namespace-pod pairs, separated by spaces (inside each pair and between the pairs). Now we turn to the bash to call `kubectl delete pods`. But before that another little step, let's separate the pairs with newlines. Xargs is our friend:

```bash
$ kubectl get pods --all-namespaces \
   -o jsonpath='{range .items[?(.spec.nodeName=="gke-cluster-1-b4c97d4d-node-psh2")]}{@.metadata.namespace} {.metadata.name} {end}' | \
   xargs -n 2
default busybox-694uu kube-system
fluentd-cloud-logging-gke-cluster-1-b4c97d4d-node-psh2
```

We use xargs another time to call `kubectl delete pods` for each of those lines. Note that xargs is not good in calling commands with multiple parameters, especially if you want to stay compatible with BSD (on Mac) and GNU xargs (on Linux).

We know that namespace and pod names don't have any special character like space. Hence, we can depend the shell's text substitution without any escaping:

```bash
$ kubectl get pods --all-namespaces \
   -o jsonpath='{range .items[?(.spec.nodeName=="gke-cluster-1-b4c97d4d-node-psh2")]}{@.metadata.namespace} {.metadata.name} {end}' |\
   xargs -n 2 | xargs -I % sh -c "kubectl delete pods --namespace=%"
pod "busybox-694uu" deleted
pod "fluentd-cloud-logging-gke-cluster-1-b4c97d4d-node-psh2" deleted
```

The `-I %` tells xargs to replace the `%` character with the string `"default busybox-694uu"`. The `sh -c` will force re-evalation of the resulting command line, leading to the desired result of calling `kubectl delete pods --namespace=default busybox-694uu`.

Notice, that it take a few seconds until pods with graceful termination timeouts actually terminate:

```bash
$ kubectl get pods
NAME                       READY     STATUS          RESTARTS   AGE
busybox-1riab              0/1       Running         0          3s
busybox-7hgrz              1/1       Running         0          1h
busybox-694uu              1/1       Terminating     0          2m
```

Also note, that the replication controller behind the busybox pods already launched another pod `busybox-1riab` on another node. Perfect!

Last but not least, note that the fluentd pods is restarted. This happens because it is not a scheduled pod going through the Kubernetes scheduler. It's a static pods which is started by the kubelet directly. We could add some more logic here like looking for the `kubernetes.io/config.mirror` annotation. But as it does not harm killing it we will keep the script as simple as it is.