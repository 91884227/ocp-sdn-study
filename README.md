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

## 流量進出架構

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

## 流量進出架構

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
