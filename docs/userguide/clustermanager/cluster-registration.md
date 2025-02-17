---
title: Cluster Registration
---

## Overview of cluster mode

Karmada supports both `Push` and `Pull` modes to manage the member clusters.
The main difference between `Push` and `Pull` modes is the way access to member clusters when deploying manifests.

### Push mode
Karmada control plane will access member cluster's `kube-apiserver` directly to get cluster status and deploy manifests.

### Pull mode
Karmada control plane will not access member cluster but delegate it to an extra component named `karmada-agent`.

Each `karmada-agent` serves for a cluster and take responsibility for:
- Register cluster to Karmada(creates the `Cluster` object)
- Maintains cluster status and reports to Karmada(updates the status of `Cluster` object)
- Watch manifests from Karmada execution space(namespace, `karmada-es-<cluster name>`) and deploy to the cluster which serves for.

## Register cluster with 'Push' mode

You can use the [kubectl-karmada](../../installation/install-cli-tools.md) CLI to `join`(register) and `unjoin`(unregister) clusters.

### Register cluster by CLI

Join cluster with name `member1` to Karmada by using the following command.
```
kubectl karmada join member1 --kubeconfig=<karmada kubeconfig> --cluster-kubeconfig=<member1 kubeconfig>
```
Repeat this step to join any additional clusters.

The `--kubeconfig` specifies the Karmada's `kubeconfig` file and the CLI infers `karmada-apiserver` context
from the `current-context` field of the `kubeconfig`. If there are more than one context is configured in
the `kubeconfig` file, it is recommended to specify the context by the `--karmada-context` flag. For example:
```
kubectl karmada join member1 --kubeconfig=<karmada kubeconfig> --karmada-context=karmada --cluster-kubeconfig=<member1 kubeconfig>
```

The `--cluster-kubeconfig` specifies the member cluster's `kubeconfig` and the CLI infers the member cluster's context
by the cluster name. If there is more than one context is configured in the `kubeconfig` file, or you don't want to use
the context name to register, it is recommended to specify the context by the `--cluster-context` flag. For example:

```
kubectl karmada join member1 --kubeconfig=<karmada kubeconfig> --karmada-context=karmada \
--cluster-kubeconfig=<member1 kubeconfig> --cluster-context=member1
```
> Note: The registering cluster name can be different from the context with `--cluster-context` specified.

### Check cluster status

Check the status of the joined clusters by using the following command.
```
kubectl get clusters

NAME      VERSION   MODE   READY   AGE
member1   v1.20.7   Push   True    66s
```
### Unregister cluster by CLI

You can unjoin clusters by using the following command.
```
kubectl karmada unjoin member1 --kubeconfig=<karmada kubeconfig> --cluster-kubeconfig=<member1 kubeconfig>
```
During unjoin process, the resources propagated to `member1` by Karmada will be cleaned up.
And the `--cluster-kubeconfig` is used to clean up the secret created at the `join` phase.

Repeat this step to unjoin any additional clusters.

## Register cluster with 'Pull' mode

### Register cluster by CLI

`karmadactl register` is used to register member clusters to the Karmada control plane with PULL mode. 
Be different from the `karmadactl join` which registers a cluster with `Push` mode, `karmadactl register` registers a cluster to Karmada control plane with `Pull` mode.

> Note: currently it only supports the Karmada control plane that was installed by `karmadactl init`.

#### Create bootstrap token in Karmada control plane

In Karmada control plane, we can use `karmadactl token create` command to create bootstrap tokens whose default ttl is 24h.

```
$ karmadactl token create --print-register-command --kubeconfig /etc/karmada/karmada-apiserver.config
```

```
# The example output is shown below
karmadactl register 10.10.x.x:32443 --token t2jgtm.9nybj0526mjw1jbf --discovery-token-ca-cert-hash sha256:f5a5a43869bb44577dba582e794c3e3750f2050d62f1b1dc80fd3d6a371b6ed4
```

More details about `bootstrap token` please refer to:
- [authenticating with bootstrap tokens](https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/)

#### Execute `karmadactl register` in the member clusters

In the Kubernetes control plane of member clusters, we also need the `kubeconfig` file of the member cluster, then directly execute the above output `karmadactl register` command.

```
$ karmadactl register 10.10.x.x:32443 --token t2jgtm.9nybj0526mjw1jbf --discovery-token-ca-cert-hash sha256:f5a5a43869bb44577dba582e794c3e3750f2050d62f1b1dc80fd3d6a371b6ed4
```

```
# The example output is shown below
[preflight] Running pre-flight checks
[prefligt] All pre-flight checks were passed
[karmada-agent-start] Waiting to perform the TLS Bootstrap
[karmada-agent-start] Waiting to construct karmada-agent kubeconfig
[karmada-agent-start] Waiting the necessary secret and RBAC
[karmada-agent-start] Waiting karmada-agent Deployment
W0825 11:03:12.167027   29336 check.go:52] pod: karmada-agent-5d659b4746-wn754 not ready. status: ContainerCreating
......
I0825 11:04:06.174110   29336 check.go:49] pod: karmada-agent-5d659b4746-wn754 is ready. status: Running

cluster(member3) is joined successfully
```

> Note: if you don't set `--cluster-name` option, it will use the cluster of current-context of the `kubeconfig` file by default.


After `karmada-agent` be deployed, it will register cluster automatically at the start-up phase.

### Check cluster status

Check the status of the registered clusters by using the same command above.
```
kubectl get clusters
NAME      VERSION   MODE   READY   AGE
member3   v1.20.7   Pull   True    66s
```
### Unregister cluster

Undeploy the `karmada-agent` and then remove the `cluster` manually from Karmada.
```
kubectl delete cluster member3
```
