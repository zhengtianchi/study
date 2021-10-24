# k8s权限&&incluster

# 权限分析

- 用户给出最高权限：直接使用最高权限，没有任何问题
- 用户不给最高权限：
  - 因为需要使用到istio的功能，因此权限应该不低于istio
  - 可能会操作一些crd，需要额外配置一些权限
  - 需要作用到cluster级别的资源，因此不能使用role，要使用clusterrole

## k8s权限

**hsmesh**需要的api-resources：

- clusterrole
- clusterrolebinding
- serviceaccount
- secret

hsmesh可以指定启动方式为`outofcluster/incluster`，使用`outofcluster`时，需要指定一个外部的`kubeconfig`文件作为配置文件来管理k8s资源，使用`incluster`的模式时，hsmesh会使用`namespace=hsmesh`的`serviceaccount`作为权限认证，一般来说，默认的sa权限不足以支持hsmesh管理资源，需要手动配置更高权限的clusterrole，clusterrolebinding和sa。

## incluster

**hsmesh**对所在k8s集群的权限不低于istiod，最高为cluster-admin的权限，即

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: cluster-admin
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
- nonResourceURLs:
  - '*'
  verbs:
  - '*'
```



在用户不允许给最高权限的情况下，需要配置一个最低权限，与istiod的权限相同：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: istiod-{{ .Values.global.istioNamespace }}
  labels:
    app: istiod
    release: {{ .Release.Name }}
rules:
  # sidecar injection controller
  - apiGroups: ["admissionregistration.k8s.io"]
    resources: ["mutatingwebhookconfigurations"]
    verbs: ["get", "list", "watch", "update", "patch"]

  # configuration validation webhook controller
  - apiGroups: ["admissionregistration.k8s.io"]
    resources: ["validatingwebhookconfigurations"]
    verbs: ["get", "list", "watch", "update"]

  # istio configuration
  # removing CRD permissions can break older versions of Istio running alongside this control plane (https://github.com/istio/istio/issues/29382)
  # please proceed with caution
  - apiGroups: ["config.istio.io", "security.istio.io", "networking.istio.io", "authentication.istio.io", "rbac.istio.io"]
    verbs: ["get", "watch", "list"]
    resources: ["*"]
  - apiGroups: ["networking.istio.io"]
    verbs: [ "get", "watch", "list", "update", "patch", "create", "delete" ]
    resources: [ "workloadentries" ]
  - apiGroups: ["networking.istio.io"]
    verbs: [ "get", "watch", "list", "update", "patch", "create", "delete" ]
    resources: [ "workloadentries/status" ]

  # auto-detect installed CRD definitions
  - apiGroups: ["apiextensions.k8s.io"]
    resources: ["customresourcedefinitions"]
    verbs: ["get", "list", "watch"]

  # discovery and routing
  - apiGroups: [""]
    resources: ["pods", "nodes", "services", "namespaces", "endpoints"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["discovery.k8s.io"]
    resources: ["endpointslices"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses", "ingressclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses/status"]
    verbs: ["*"]

  # required for CA's namespace controller
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["create", "get", "list", "watch", "update"]

  # Istiod and bootstrap.
  - apiGroups: ["certificates.k8s.io"]
    resources:
      - "certificatesigningrequests"
      - "certificatesigningrequests/approval"
      - "certificatesigningrequests/status"
    verbs: ["update", "create", "get", "delete", "watch"]
  - apiGroups: ["certificates.k8s.io"]
    resources:
      - "signers"
    resourceNames:
    - "kubernetes.io/legacy-unknown"
    verbs: ["approve"]

  # Used by Istiod to verify the JWT tokens
  - apiGroups: ["authentication.k8s.io"]
    resources: ["tokenreviews"]
    verbs: ["create"]

  # Used by Istiod to verify gateway SDS
  - apiGroups: ["authorization.k8s.io"]
    resources: ["subjectaccessreviews"]
    verbs: ["create"]

  # Use for Kubernetes Service APIs
  - apiGroups: ["networking.x-k8s.io"]
    resources: ["*"]
    verbs: ["get", "watch", "list"]

  # Needed for multicluster secret reading, possibly ingress certs in the future
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "watch", "list"]
---
```

由于istiod本身不会操作`Deployment`等资源，因此总体而言hsmesh的权限应该要高于istio，具体请参考

[hsmesh rbac](https://gitlab.hundsun.com/ORCA1.0/Mesh/docs/tree/master/mesh-docs/mesh-server/rbac)

# 参考

> verbs 参考：
>
> https://kubernetes.io/docs/reference/access-authn-authz/authorization/#determine-the-request-verb

>apiGroup 参考：
>
>`kubectl api-resources`

> resources 参考：
>
> https://kubernetes.io/docs/reference/kubernetes-api/