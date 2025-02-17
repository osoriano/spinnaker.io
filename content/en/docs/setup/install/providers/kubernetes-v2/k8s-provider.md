---
title:  "Kubernetes Provider"
linkTitle: "K8s Provider"
description: >
  Spinnaker's Kubernetes provider fully supports Kubernetes-native, manifest-based deployments and is the recommended provider for deploying to Kubernetes with Spinnaker.
---

Spinnaker's Kubernetes provider fully supports Kubernetes-native, manifest-based deployments and is the recommended provider for deploying to Kubernetes with Spinnaker.
[Spinnaker's legacy Kubernetes provider](https://www.spinnaker.io/setup/install/providers/kubernetes/)
is [scheduled for removal](https://github.com/spinnaker/governance/blob/master/rfc/eol_kubernetes_v1.md) in Spinnaker 1.21.

## Accounts

A Spinnaker [Account](/docs/concepts/providers/#accounts) maps to a
credential that can authenticate against your Kubernetes Cluster.

## Prerequisites

The Kubernetes provider has two requirements:

* A [kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) file

    The `kubeconfig` file allows Spinnaker to authenticate against your cluster
    and to have read/write access to any resources you expect it to manage. You
    can think of it as private key file to let Spinnaker connect to your cluster.
    You can request this from your Kubernetes cluster administrator.

* [kubectl](https://kubernetes.io/docs/user-guide/kubectl/) CLI tool

    Spinnaker relies on `kubectl` to manage all API access. It's installed
    along with Spinnaker.

    Spinnaker also relies on `kubectl` to access your Kubernetes cluster; only
    `kubectl` fully supports many aspects of the Kubernetes API, such as 3-way
    merges on `kubectl apply`, and API discovery. Though this creates a
    dependency on a binary, the good news is that any authentication method or
    API resource that `kubectl` supports is also supported by Spinnaker. This
    is an improvement over the original Kubernetes provider in Spinnaker.


<span class="begin-collapsible-section"></span>

### Optional: Create a Kubernetes Service Account

If you want, you can associate Spinnaker with a [Kubernetes Service
Account](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/),
even when managing multiple Kubernetes clusters. This can be useful if you need
to grant Spinnaker certain roles in the cluster later on, or you typically
depend on an authentication mechanism that doesn't work in all environments.

Given that you want to create a Service Account in existing context `$CONTEXT`,
the following commands will create `spinnaker-service-account`, and add its
token under a new user called `${CONTEXT}-token-user` in context `$CONTEXT`.

```bash
CONTEXT=$(kubectl config current-context)

# This service account uses the ClusterAdmin role -- this is not necessary,
# more restrictive roles can by applied.
kubectl apply --context $CONTEXT \
    -f https://spinnaker.io/downloads/kubernetes/service-account.yml

TOKEN=$(kubectl get secret --context $CONTEXT \
   $(kubectl get serviceaccount spinnaker-service-account \
       --context $CONTEXT \
       -n spinnaker \
       -o jsonpath='{.secrets[0].name}') \
   -n spinnaker \
   -o jsonpath='{.data.token}' | base64 --decode)

kubectl config set-credentials ${CONTEXT}-token-user --token $TOKEN

kubectl config set-context $CONTEXT --user ${CONTEXT}-token-user
```

<span class="end-collapsible-section"></span>

<span class="begin-collapsible-section"></span>

### Optional: Configure Kubernetes roles (RBAC)

If your Kubernetes cluster supports
[RBAC](https://kubernetes.io/docs/admin/authorization/rbac/)
and you want to restrict permissions granted to your Spinnaker account, you
will need to follow the below instructions.

The following YAML creates the correct `ClusterRole`, `ClusterRoleBinding`, and
`ServiceAccount`. If you limit Spinnaker to operating on an explicit list of
namespaces (using the `namespaces` option), you need to use `Role` &
`RoleBinding` instead of `ClusterRole` and `ClusterRoleBinding`, and apply the
`Role` and `RoleBinding` to each namespace Spinnaker manages. You can read
about the difference between `ClusterRole` and `Role`
[here](https://kubernetes.io/docs/admin/authorization/rbac/#rolebinding-and-clusterrolebinding).
If you're using RBAC to restrict the Spinnaker service account to a particular namespace,
you must specify that namespace when you add the account to Spinnaker.
If you don't specify any namespaces, then Spinnaker will attempt to list all namespaces,
which requires a cluster-wide role. Without a cluster-wide role configured
and specified namespaces, you will see deployment
[timeouts in the "Wait for Manifest to Stabilize" task](https://github.com/spinnaker/spinnaker/issues/3666#issuecomment-485001361).

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
 name: spinnaker-role
rules:
- apiGroups: [""]
  resources: ["namespaces", "configmaps", "events", "replicationcontrollers", "serviceaccounts", "pods/log"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods", "services", "secrets"]
  verbs: ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"]
- apiGroups: ["autoscaling"]
  resources: ["horizontalpodautoscalers"]
  verbs: ["list", "get"]
- apiGroups: ["apps"]
  resources: ["controllerrevisions"]
  verbs: ["list"]
- apiGroups: ["extensions", "apps"]
  resources: ["daemonsets", "deployments", "deployments/scale", "ingresses", "replicasets", "statefulsets"]
  verbs: ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"]
# These permissions are necessary for halyard to operate. We use this role also to deploy Spinnaker itself.
- apiGroups: [""]
  resources: ["services/proxy", "pods/portforward"]
  verbs: ["create", "delete", "deletecollection", "get", "list", "patch", "update", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
 name: spinnaker-role-binding
roleRef:
 apiGroup: rbac.authorization.k8s.io
 kind: ClusterRole
 name: spinnaker-role
subjects:
- namespace: spinnaker
  kind: ServiceAccount
  name: spinnaker-service-account
---
apiVersion: v1
kind: ServiceAccount
metadata:
 name: spinnaker-service-account
 namespace: spinnaker
```

<span class="end-collapsible-section"></span>

<span class="begin-collapsible-section"></span>

## Migrating from Spinnaker's legacy Kubernetes provider

> Prior to the deprecation of Spinnaker's legacy (V1) Kubernetes provider, the
> standard provider was often referred to as the V2 provider. For clarity, this
> section refers to the providers as the V1 and V2 providers.

There is no automatic pipeline migration from the V1 provider to V2, for a few
reasons:

* Unlike the V1 provider, the V2 provider encourages you to store your
  Kubernetes Manifests outside of Spinnaker in some versioned, backing storage,
  such as Git or GCS.

* The V2 provider encourages you to leverage the Kubernetes native deployment
  orchestration (e.g.
  [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/))
  instead of the Spinnaker red/black, where possible.

* The initial operations available on Kubernetes manifests (e.g. scale, pause
  rollout, delete) in the V2 provider don't map nicely to the operations in the
  V1 provider unless you contort Spinnaker abstractions to match those of
  Kubernetes. To avoid building dense and brittle mappings between Spinnaker's
  logical resources and Kubernetes's infrastructure resources, we chose to
  adopt the Kubernetes resources and operations more natively.

* The V2 provider does __not__ use the [Docker Registry
  Provider](https://www.spinnaker.io/setup/install/providers/docker-registry/).
  You may still need Docker Registry accounts to trigger pipelines, but
  otherwise we encourage you to stop using Docker Registry accounts in Spinnaker.
  The V2 provider requires that you manage your private registry [configuration
  and authentication](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)
  yourself.  

However, you can easily migrate your _infrastructure_ into the V2 provider.
For any V1 account you have running, you can add a V2 account following the
steps [below](#adding-an-account). This will surface your infrastructure twice
(once per account) helping your pipeline & operation migration.

![A V1 and V2 provider surfacing the same infrastructure](/docs/setup/install/providers/kubernetes-v2/v1v2.png)

<span class="end-collapsible-section"></span>

## Adding an account

First, make sure that the provider is enabled:

```bash
hal config provider kubernetes enable
```

Then add the account:

```bash
CONTEXT=$(kubectl config current-context)

hal config provider kubernetes account add my-k8s-account \
    --context $CONTEXT
```

Finally, enable [artifact support](/docs/reference/artifacts/#enabling-artifact-support).

## Advanced account settings

If you're looking for more configurability, please see the other options listed
in the [Halyard
Reference](/docs/reference/halyard/commands#hal-config-provider-kubernetes-account-add).
