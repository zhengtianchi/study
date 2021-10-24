# 多云管理FAQ

## CNI支持

**目前只支持flannel/calico**

使用flannel可能会导致测试不通过，但是服务发现正常，需要等待一段时间，flannel才能正常工作。

calico一切正常。

## 多网络环境

指pod-to-pod网络不通，但是宿主机网络连通。


## 多主环境，集群间无法正常请求

检查remote-secret是否正常创建。
多主情况下，remote-secret非常重要，是服务发现最关键的一步。

## hsmesh生成的证书不可用问题

istiod的pod时间不准，会导致新生成的根证书可使用时间在istiod初始化时间之后，导致不可用，等待一段时间后即可。

## 时间问题会导致证书不可用：

日志描述：

> current time 2021-06-11T01:40:31Z is before 2021-06-11T01:53:26Z"

解决方案：

```bash
timedatectl set-timezone Asia/Shanghai
yum -y install ntp
ntpdate <amend time ip>
```

> <amend time ip>中请填写实际的校准时间IP
>
> 公司内提供了一些校准时间的ip：
>
> - 192.168.60.14
> - 192.168.60.11
> - 192.168.60.10

## 根证书必须使用同一个，二级证书需要重新签发

可能存在证书过期问题，需要进一步调研
使用istioOperator安装默认使用的证书不可用与多云的次级证书签发

## 主从模式下，从集群无法主动向主集群发起请求

v2.3.1版本遗留问题:

同clusterDomain下，主从集群可以互相通信，当主从集群不同clusterDomain时，从集群无法正常服务发现到主集群。

## 控制面使用的nodeport模式，从集群的服务发现地址填什么

istio官网：

> Save the address of cluster1’s east-west gateway.
> ```bash
> export DISCOVERY_ADDRESS=$(kubectl \
>     --context="${CTX_CLUSTER1}" \
>     -n istio-system get svc istio-eastwestgateway \
>     -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
> ```

需要将控制面的eastwestgateway的EXTERNAL-IP作为服务发现地址，但是一般情况下国内使用的是clusterIP。

解决方法：

```bash
export DISCOVERY_ADDRESS=$(kubectl --kubeconfig=your/multiconf --context="${CTX_CLUSTER1}" get svc istio-eastwestgateway -n istio-system -o jsonpath='{.spec.clusterIP}')
```

使用eastwestgateway的Cluster IP作为服务发现地址，但是不可用podIP，podIP可能会产生漂移会导致请求不通从而导致服务发现失败。

## 多网络环境相较于单网络有什么区别

多网络环境中，不需要手动添加pod-to-pod网络的网关，但是需要在从集群中添加组集群的clusterIP的网关。

**需要手动配置istio的configmap，保证集群间通信可以走东西向网关**

## 如何获取当前集群的clusterDomain（dnsDomain）

```bash
kubectl get cm coredns -n kube-system -oyaml|grep kubernetes | awk '{print $2}'
```

