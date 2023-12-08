# ocp-sdn-study

# 背景知識

OCP 的`SDN`基本上由 `iptable` + `Open vSwitch` + `Network Namespace`實現

## IPTables 的 Target

```yaml
user-defined chain:
special values:
    ACCEPT
    DROP
    QUEUE
    RETURN: stop traversing this chain and resume at the next rule in the previous (calling) chain.
    MASQUERADE:
        只可在 nat table 的 POSTROUTING chain使用
    DNAT: the destination address of the packet should be modified
        只可在 nat table 的 POSTROUTING 和 OUTPUT chain使用
        par: --to-destination ipaddr[-ipaddr][:port-port]
```

# 網路實現: POD 到 local Pod (兩個 pod 在同一個 Node 上)

## 環境配置

```yaml
podA: 10.131.0.15
podB: 10.131.0.18
```

## 流量進出路徑

以 `where: in/out` 表示

```yaml
podA: / eth0
ovs-bridge: port 16 / port 19 (0x13)
podB: eth0 /
```

## ovs-bridge 流量進出規則分析

```yaml
Table 00:
    priority=100,ip
    actions=goto_table:20
Table 20:
    priority=100,ip
    in_port=16
    nw_src=10.131.0.15
    actions=load:0xc888a8->NXM_NX_REG0[],
    goto_table:27
Table 27:
    priority=50
    reg0=0xc888a8
    actions=goto_table:30
Table 30:
    priority=0
    actions=goto_table:31
Table 31:
    priority=200,ip
    nw_dst=10.131.0.0/23 (10.131.0.0 - 10.131.1.255)
    actions=goto_table:70
Table 70:
    priority=100,ip
    nw_dst=10.131.0.18
    actions:
        load:0xc888a8->NXM_NX_REG1[]
        load:0x13->NXM_NX_REG2[]
    goto_table:80
Table 80:
    priority=50
    reg1=0xc888a8
    actions=output:NXM_NX_REG2[]
```

# 網路實現: Node 到 local Pod

## 環境配置

```yaml
pod: 10.131.0.15 (Node:10.250.133.44)
```

## 流量進出路徑

以 `where: in/out` 表示

```yaml
node: / tun0
ovs-bridge: port 2 / port 16
pod: eth0 /
```

## node routeTable 設置

```yaml
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.128.0.0      0.0.0.0         255.252.0.0     U     0      0        0 tun0
```

## ovs-bridge 設置

```yaml
OFPST_PORT_DESC reply (OF1.3) (xid=0x3):
  2(tun0): addr:22:db:52:3b:f5:8b
  16(vethf31d7c7b): addr:6e:ba:9e:f1:8f:2f
```

## ovs-bridge Table 分析

```yaml
[Table 70]:
    priority=100,ip
    nw_dst=10.131.0.15
    actions:
        load:0xc888a8->NXM_NX_REG1
        load:0x10->NXM_NX_REG2
    goto_table:80
[Table 80]:
    priority=50
    reg1=0xc85a93
    actions=output:NXM_NX_REG2[]
```

# external traffic 透過 NodePort 到 Pod

## 環境配置

```yaml
node-port: 30007
service對應到的pod: 10.128.2.47、10.131.0.77
service對應到的pod的port: 8080
```

## iptable 分析

```yaml
[Chain]PREROUTING: [Target]KUBE-SERVICES
[Chain]KUBE-SERVICES: [Target]KUBE-NODEPORTS
[Chain]KUBE-NODEPORTS: [Target]KUBE-EXT-HU4HVRCAUIB5YQHI (符合 tcp dpt:30007)
[Chain]KUBE-EXT-HU4HVRCAUIB5YQHI: [Target]KUBE-SVC-HU4HVRCAUIB5YQHI
[Chain]KUBE-SVC-HU4HVRCAUIB5YQHI:
    [Target]:KUBE-SEP-TKINKZP2A5JDJDZN
    [Target]:KUBE-SEP-2G7MVSJPPKG6GFCF
[Chain]KUBE-SEP-TKINKZP2A5JDJDZN: [Target]DNAT (to 10.128.2.47:8080)
[Chain]KUBE-SEP-2G7MVSJPPKG6GFCF: [Target]DNAT (to:10.131.0.77:8080)
```

# Pod 到 Service

## 環境配置

```yaml
pod: 10.131.0.15 (Node:10.250.133.44)
service: httpd-service (172.30.100.234)
    pod-s1: 10.128.2.43 (Node:10.250.133.45)
    pod-s2: 10.131.0.21 (Node:10.250.133.44)
    pod-s3: 10.131.0.18 (Node:10.250.133.44)
```

## 流量進出路徑

以 `where: in/out` 表示

```yaml
pod:  / eth0
ovs-bridge:  port 16(vethf31d7c7b) / port 2(tun0)
network host: tun0 -> tun0
    iptable:
        OUTPUT: 換掉Des (172.30.100.234(service IP) -> 10.131.0.21(pod-s2 IP))
        POSTROUTING: 換掉Source (10.131.0.15(pod IP) -> 10.250.133.44(Node IP))
ovs-bridge: 2(tun0) / 22(vethe39b1565)
pod-s2: eth0 /
```

## ovs-bridge (port 16 -> port 2)進出分析

```yaml
[table 31]: priority=100,ip
  nw_dst=172.30.0.0/16
  actions=goto_table:60
[table 60]: priority=200
  actions=output:2
```

## iptable 分析

```yaml
[Chain]OUTPUT: [Target]KUBE-SERVICES
[Chain]KUBE-SERVICES: [Target]KUBE-SVC-GAUART32FUYN4NZ7 (符合 tcp --dport 80)
[Chain]KUBE-SVC-GAUART32FUYN4NZ7:
    [Target]KUBE-SEP-6WMDVDAGBKYAF532
    [Target]KUBE-SEP-CHWJ3B6W2ZHIFOS3
    [Target]KUBE-SEP-OCJSQJICVLUKUFP2
[Chain]KUBE-SEP-6WMDVDAGBKYAF532: [Target]DNAT (to 10.128.2.43:8080)
[Chain]KUBE-SEP-CHWJ3B6W2ZHIFOS3: [Target]DNAT (to 10.131.0.18:8080)
[Chain]KUBE-SEP-OCJSQJICVLUKUFP2: [Target]DNAT (to 10.131.0.21:8080)
```
