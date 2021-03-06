+++
categories = []
date = "2020-02-22"
description = ""
slug = ""
tags = []
title = "Set resource limits"
draft = false
toc = true
weight = 80
+++

### Introduction
In this exercise, we cover:

 - How to set resource limits
 - What can happen if you don't

_Note:_ This exercise can be a little finicky in cloud environments.
And be careful that you don't request too much memory if your cluster is running on your machine and doesn't enforce overall memory limits.

### Setup
In this example, we'll use an app with an exploitable
memory exhaustion denial of service. This will be fun to watch...

We'll use something sort of like a "Billion Laughs"-style attack. Here's the example
from [Kubernetes CVE-2019-11253](https://github.com/kubernetes/kubernetes/issues/83253):
```yaml
apiVersion: v1
data:
  a: &a ["web","web","web","web","web","web","web","web","web"]
  b: &b [*a,*a,*a,*a,*a,*a,*a,*a,*a]
  c: &c [*b,*b,*b,*b,*b,*b,*b,*b,*b]
  d: &d [*c,*c,*c,*c,*c,*c,*c,*c,*c]
  e: &e [*d,*d,*d,*d,*d,*d,*d,*d,*d]
  f: &f [*e,*e,*e,*e,*e,*e,*e,*e,*e]
  g: &g [*f,*f,*f,*f,*f,*f,*f,*f,*f]
  h: &h [*g,*g,*g,*g,*g,*g,*g,*g,*g]
  i: &i [*h,*h,*h,*h,*h,*h,*h,*h,*h]
kind: ConfigMap
metadata:
  name: yaml-bomb
  namespace: default
```

The example app just allocates memory based on the value of the parameter you give it.

View the code:

```
less static/memory-exploder/main.go
```

{{< embedCodeFile file="/static/memory-exploder/main.go" language="go" >}}

Then deploy:
```
kubectl apply -f https://securek8s.dev/memory-exploder/buggy.yaml
```

### "Attack"
Call the bad method:
```bash
curl -X POST "http://${WORKSHOP_NODE_IP:-localhost}:31304/1234"
```

This will work. But bump up the number and it will start getting bad!

```bash
curl -X POST "http://${WORKSHOP_NODE_IP:-localhost}:31304/123456789012"
```

You can exit from the request and try to run:
```
kubectl top pod -n buggy
```
to see the pod fall over, if your cluster has Heapster installed.
(It's a bit of a race!)

On Docker for Mac you may have more luck with:

```
docker stats
```

(Note: for the purposes of the workshop, we won't try to make
nodes completely fall over...)

### Countermeasure
Apply a memory and CPU limit.

See the diff:

```
kubectl diff -f https://securek8s.dev/memory-exploder/buggy-but-limited.yaml
```

Then deploy:

```
kubectl diff -f https://securek8s.dev/memory-exploder/buggy-but-limited.yaml
```

### Attack effects after patching
The app gets OOMKilled and restarted instead of causing nodes to
become unstable or crash.

You can see a fresh pod being created after your request:

```
kubectl get pod -n buggy
```

The app will hang up early if you make the same large request from earlier:

```bash
curl -X POST "http://${WORKSHOP_NODE_IP:-localhost}:31304/123456789012"
```

If you check your pods again, you'll see that the container restarted:

```
# kubectl get pod -n buggy
NAME                     READY   STATUS    RESTARTS   AGE
buggy-6659cc6895-29j98   1/1     Running   1          2m50s
```

### How to use it yourself
Add requests and limits to each of your pods.

Be careful about what you choose, especially for limits.
Too low a CPU limit can interfere with app functionality, especially if you do things like RSA.
Too low a memory limit will get your app OOMKilled repeatedly.

### Next up
All done for now! 🙂

<!--
We'll cover effective metadata in the next exercise:

[**Bonus: Apply good, consistent metadata**](../90-metadata)
-->
