# kube-proxy

每台机器上都运行一个 kube-proxy 服务，它监听 API server 中 service 和 endpoint 的变化情况，并通过 iptables 等来为服务配置负载均衡（仅支持 TCP 和 UDP）。

> **注意**: 从 Kubernetes 1.33 开始，Endpoints API 已被弃用，kube-proxy 将逐步迁移到使用 EndpointSlices API 来获取更好的性能和功能支持。

kube-proxy 可以直接运行在物理机上，也可以以 static pod 或者 daemonset 的方式运行。

kube-proxy 当前支持以下几种实现

* userspace：最早的负载均衡方案，它在用户空间监听一个端口，所有服务通过 iptables 转发到这个端口，然后在其内部负载均衡到实际的 Pod。该方式最主要的问题是效率低，有明显的性能瓶颈。
* iptables：传统的负载均衡方案，完全以 iptables 规则的方式来实现 service 负载均衡。该方式最主要的问题是在服务多的时候产生太多的 iptables 规则，非增量式更新会引入一定的时延，大规模情况下有明显的性能问题
* ipvs：为解决 iptables 模式的性能问题，v1.11 新增了 ipvs 模式（v1.8 开始支持测试版，并在 v1.11 GA），采用增量式更新，并可以保证 service 更新期间连接保持不断开
* nftables：新推出的高性能模式（v1.29 开始支持 alpha 版，目前处于 beta 状态，预计 v1.33 GA），通过使用 "verdict map" 实现更高效的数据包路由，将包处理延迟从 O(n) 降低到大致 O(1)，显著减少了首包延迟，特别是在大规模集群中。需要 Linux 内核 5.13 或更新版本
* winuserspace：同 userspace，但仅工作在 windows 节点上

注意：
- 使用 ipvs 模式时，需要预先在每台 Node 上加载内核模块 `nf_conntrack_ipv4`, `ip_vs`, `ip_vs_rr`, `ip_vs_wrr`, `ip_vs_sh` 等。
- 使用 nftables 模式时，需要 Linux 内核 5.13 或更新版本，可能与某些现有网络组件不完全兼容，目前还不是默认模式。

```bash
# load module <module_name>
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4

# to check loaded modules, use
lsmod | grep -e ip_vs -e nf_conntrack_ipv4
# or
cut -f1 -d " "  /proc/modules | grep -e ip_vs -e nf_conntrack_ipv4
```

## Iptables 示例

### Kube-proxy iptables 示意图

![](../../.gitbook/assets/iptables-mode%20%281%29.png)

\(图片来自[cilium/k8s-iptables-diagram](https://github.com/cilium/k8s-iptables-diagram)\)

### Kube-proxy NAT 示意图

![](../../.gitbook/assets/kube-proxy-nat-flow.png)

（图片来自[kube-proxy iptables "nat" control flow](https://docs.google.com/drawings/d/1MtWL8qRTs6PlnJrW4dh8135_S9e2SaawT410bJuoBPk/edit)）

### Iptables 示例

```bash
# Masquerade
-A KUBE-MARK-DROP -j MARK --set-xmark 0x8000/0x8000
-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE

# clusterIP and publicIP
-A KUBE-SERVICES ! -s 10.244.0.0/16 -d 10.98.154.163/32 -p tcp -m comment --comment "default/nginx: cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.98.154.163/32 -p tcp -m comment --comment "default/nginx: cluster IP" -m tcp --dport 80 -j KUBE-SVC-4N57TFCL4MD7ZTDA
-A KUBE-SERVICES -d 12.12.12.12/32 -p tcp -m comment --comment "default/nginx: loadbalancer IP" -m tcp --dport 80 -j KUBE-FW-4N57TFCL4MD7ZTDA

# Masq for publicIP
-A KUBE-FW-4N57TFCL4MD7ZTDA -m comment --comment "default/nginx: loadbalancer IP" -j KUBE-MARK-MASQ
-A KUBE-FW-4N57TFCL4MD7ZTDA -m comment --comment "default/nginx: loadbalancer IP" -j KUBE-SVC-4N57TFCL4MD7ZTDA
-A KUBE-FW-4N57TFCL4MD7ZTDA -m comment --comment "default/nginx: loadbalancer IP" -j KUBE-MARK-DROP

# Masq for nodePort
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/nginx:" -m tcp --dport 30938 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/nginx:" -m tcp --dport 30938 -j KUBE-SVC-4N57TFCL4MD7ZTDA

# load balance for each endpoints
-A KUBE-SVC-4N57TFCL4MD7ZTDA -m comment --comment "default/nginx:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-UXHBWR5XIMVGXW3H
-A KUBE-SVC-4N57TFCL4MD7ZTDA -m comment --comment "default/nginx:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-TOYRWPNILILHH3OR
-A KUBE-SVC-4N57TFCL4MD7ZTDA -m comment --comment "default/nginx:" -j KUBE-SEP-6QCC2MHJZP35QQAR

# endpoint #1
-A KUBE-SEP-6QCC2MHJZP35QQAR -s 10.244.3.4/32 -m comment --comment "default/nginx:" -j KUBE-MARK-MASQ
-A KUBE-SEP-6QCC2MHJZP35QQAR -p tcp -m comment --comment "default/nginx:" -m tcp -j DNAT --to-destination 10.244.3.4:80

# endpoint #2
-A KUBE-SEP-TOYRWPNILILHH3OR -s 10.244.2.4/32 -m comment --comment "default/nginx:" -j KUBE-MARK-MASQ
-A KUBE-SEP-TOYRWPNILILHH3OR -p tcp -m comment --comment "default/nginx:" -m tcp -j DNAT --to-destination 10.244.2.4:80

# endpoint #3
-A KUBE-SEP-UXHBWR5XIMVGXW3H -s 10.244.1.2/32 -m comment --comment "default/nginx:" -j KUBE-MARK-MASQ
-A KUBE-SEP-UXHBWR5XIMVGXW3H -p tcp -m comment --comment "default/nginx:" -m tcp -j DNAT --to-destination 10.244.1.2:80
```

如果服务设置了 `externalTrafficPolicy: Local` 并且当前 Node 上面没有任何属于该服务的 Pod，那么在 `KUBE-XLB-4N57TFCL4MD7ZTDA` 中会直接丢掉从公网 IP 请求的包：

```bash
-A KUBE-XLB-4N57TFCL4MD7ZTDA -m comment --comment "default/nginx: has no local endpoints" -j KUBE-MARK-DROP
```

## ipvs 示例

[Kube-proxy IPVS mode](https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/ipvs/README.md) 列出了各种服务在 IPVS 模式下的工作原理。

![](../../.gitbook/assets/ipvs-mode.png)

```bash
$ ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.0.0.1:443 rr persistent 10800
  -> 192.168.0.1:6443             Masq    1      1          0
TCP  10.0.0.10:53 rr
  -> 172.17.0.2:53                Masq    1      0          0
UDP  10.0.0.10:53 rr
  -> 172.17.0.2:53                Masq    1      0          0
```

注意，IPVS 模式也会使用 iptables 来执行 SNAT 和 IP 伪装（MASQUERADE），并使用 ipset 来简化 iptables 规则的管理：

| ipset 名 | 成员 | 用途 |
| :--- | :--- | :--- |
| KUBE-CLUSTER-IP | All service IP + port | Mark-Masq for cases that `masquerade-all=true` or `clusterCIDR` specified |
| KUBE-LOOP-BACK | All service IP + port + IP | masquerade for solving hairpin purpose |
| KUBE-EXTERNAL-IP | service external IP + port | masquerade for packages to external IPs |
| KUBE-LOAD-BALANCER | load balancer ingress IP + port | masquerade for packages to load balancer type service |
| KUBE-LOAD-BALANCER-LOCAL | LB ingress IP + port with `externalTrafficPolicy=local` | accept packages to load balancer with `externalTrafficPolicy=local` |
| KUBE-LOAD-BALANCER-FW | load balancer ingress IP + port with `loadBalancerSourceRanges` | package filter for load balancer with `loadBalancerSourceRanges` specified |
| KUBE-LOAD-BALANCER-SOURCE-CIDR | load balancer ingress IP + port + source CIDR | package filter for load balancer with `loadBalancerSourceRanges` specified |
| KUBE-NODE-PORT-TCP | nodeport type service TCP port | masquerade for packets to nodePort\(TCP\) |
| KUBE-NODE-PORT-LOCAL-TCP | nodeport type service TCP port with `externalTrafficPolicy=local` | accept packages to nodeport service with `externalTrafficPolicy=local` |
| KUBE-NODE-PORT-UDP | nodeport type service UDP port | masquerade for packets to nodePort\(UDP\) |
| KUBE-NODE-PORT-LOCAL-UDP | nodeport type service UDP port with`externalTrafficPolicy=local` | accept packages to nodeport service with`externalTrafficPolicy=local` |

## 启动 kube-proxy 示例

```bash
# iptables 模式（传统模式）
kube-proxy --kubeconfig=/var/lib/kubelet/kubeconfig --cluster-cidr=10.240.0.0/12 --proxy-mode=iptables

# ipvs 模式（高性能模式）
kube-proxy --kubeconfig=/var/lib/kubelet/kubeconfig --cluster-cidr=10.240.0.0/12 --proxy-mode=ipvs

# nftables 模式（最新高性能模式，需要内核 5.13+）
kube-proxy --kubeconfig=/var/lib/kubelet/kubeconfig --cluster-cidr=10.240.0.0/12 --proxy-mode=nftables
```

## kube-proxy 工作原理

kube-proxy 监听 API server 中 service 和 endpoint（以及 EndpointSlices）的变化情况，并通过 userspace、iptables、ipvs、nftables 或 winuserspace 等 proxier 来为服务配置负载均衡（仅支持 TCP 和 UDP）。现代版本的 kube-proxy 优先使用 EndpointSlices API 而不是传统的 Endpoints API，以获得更好的性能和扩展性。

![](../../.gitbook/assets/kube-proxy%20%283%29.png)

## kube-proxy 不足

kube-proxy 目前仅支持 TCP 和 UDP，不支持 HTTP 路由，并且也没有健康检查机制。这些可以通过自定义 [Ingress Controller](../../extension/ingress/) 的方法来解决。

## nftables 模式详细说明

nftables 模式是 kube-proxy 的最新代理模式，旨在解决 iptables 模式在大规模集群中的性能问题：

### 性能优势

1. **数据平面延迟优化**：使用 "verdict map" 进行更高效的数据包路由，将包处理时间从 O(n) 降低到大致 O(1)
2. **控制平面延迟优化**：相比 iptables 模式提供更增量式的更新，更新大小仅与变更的 Services 和 endpoints 成比例
3. **显著减少首包延迟**：特别是在大规模集群中表现突出
4. **消除全局锁竞争问题**：避免了 iptables 模式的全局锁竞争

### 使用要求和限制

- 需要 Linux 内核 5.13 或更新版本
- 与 iptables 模式不是 100% 兼容
- 可能与某些现有网络组件不兼容
- 目前还不是默认模式

### 启用方法

可以通过以下方式启用 nftables 模式：

```bash
# 直接启动 kube-proxy
kube-proxy --proxy-mode nftables

# 使用 kubeadm 集群
kubeadm init --config kubeadm-config.yaml  # 需要在配置中指定 proxy-mode

# 将现有集群从 iptables/ipvs 模式转换为 nftables 模式
kubectl patch configmap kube-proxy -n kube-system --patch '{"data":{"config.conf":"mode: nftables"}}'
```

### 发展前景

- 预计在 Kubernetes v1.33 版本中达到 GA 状态
- 未来可能最终替代 IPVS 模式
- iptables 模式将继续得到支持
