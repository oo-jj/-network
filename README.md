# 酒店有线无线网络覆盖设计与搭建项目简介（含代码）

## 一、项目概述

本项目为某新建酒店网络建设项目。酒店共4栋大楼（1栋办公楼、3栋客房），要求实现全楼有线无线覆盖，办公网络可访问内网服务器和互联网，客房网络仅能访问互联网。

采用三层网络架构，核心层运行OSPF，汇聚层作为网关并启用DHCP，接入层区分有线和无线。AC旁挂核心，AP通过Option 43上线，提供Work和Hotel两个SSID。通过ACL策略禁止客房网段访问服务器。

---

## 二、核心配置代码

### 2.1 核心交换机（Core1）OSPF配置
```shell
ospf 1
 area 0.0.0.1
  network 172.31.200.0 0.0.0.3      # 与Core2互联
  network 172.31.200.4 0.0.0.3      # 与AC互联
  network 172.31.200.8 0.0.0.3      # 与防火墙互联
  network 172.31.200.12 0.0.0.3     # 与1#楼汇聚互联
  network 172.31.200.16 0.0.0.3     # 与2#楼汇聚互联
  network 172.31.200.20 0.0.0.3     # 与3#楼汇聚互联
  network 172.31.200.24 0.0.0.3     # 与4#楼汇聚互联
  network 172.31.200.60 0.0.0.3     # 与服务器核心互联
```

### 2.2 办公楼汇聚交换机（网关 + DHCP）
```shell
dhcp enable

interface Vlanif1
 ip address 172.31.1.254 255.255.255.0
 dhcp select interface
 dhcp server dns-list 172.31.100.1 114.114.114.114

interface Vlanif2
 ip address 172.31.2.254 255.255.255.0
 dhcp select interface
 dhcp server dns-list 172.31.100.1 114.114.114.114

interface Vlanif3
 ip address 172.31.3.254 255.255.255.0
 dhcp select interface
 dhcp server option 43 sub-option 3 ascii 172.31.200.254

ospf 1
 area 0.0.0.1
  network 172.31.1.0 0.0.0.255
  network 172.31.2.0 0.0.0.255
  network 172.31.3.0 0.0.0.255
  network 172.31.200.12 0.0.0.3
```

### 2.3 POE交换机（AP接入）
```shell
vlan batch 2 to 3

interface GigabitEthernet0/0/1
 port link-type trunk
 port trunk allow-pass vlan all

interface GigabitEthernet0/0/2
 port link-type trunk
 port trunk allow-pass vlan all
 port trunk pvid vlan 3

port-group 1
 group-member Ethernet 0/0/1 to Ethernet 0/0/22
 port link-type trunk
 port trunk allow-pass vlan all
 port trunk pvid vlan 3
```

### 2.4 AC无线配置
```shell
capwap source interface LoopBack0

wlan
 regulatory-domain-profile name default
  country-code CN
 ap auth-mode no-auth

 security-profile name 001
  security wpa-wpa2 psk pass-phrase work1234 aes
 ssid-profile name 001
  ssid Work
 vap-profile name 001
  forward-mode direct-forward
  service-vlan vlan-id 2
  security-profile 001
  ssid-profile 001
 ap-group name work
  vap-profile 001 wlan 1 radio all

 security-profile name 002
  security wpa-wpa2 psk pass-phrase hotel1234 aes
 ssid-profile name 002
  ssid Hotel
 vap-profile name 002
  forward-mode direct-forward
  service-vlan vlan-id 50
  security-profile 002
  ssid-profile 002
 ap-group name default
  vap-profile 002 wlan 2 radio all
```

### 2.5 防火墙配置
```shell
interface GigabitEthernet1/0/1
 ip address 172.31.200.53 255.255.255.252
 service-manage all permit

interface GigabitEthernet1/0/2
 ip address 172.31.200.10 255.255.255.252
 service-manage all permit

firewall zone trust
 add interface GigabitEthernet1/0/2
firewall zone untrust
 add interface GigabitEthernet1/0/1

security-policy
 rule name 1
  source-zone trust
  destination-zone untrust
  action permit

ip route-static 0.0.0.0 0.0.0.0 172.31.200.54
```

### 2.6 服务器核心ACL（禁止客房访问）
```shell
acl number 3001
 rule 1 deny ip destination 172.31.4.0 0.0.0.255
 rule 2 deny ip destination 172.31.5.0 0.0.0.255
 rule 3 deny ip destination 172.31.6.0 0.0.0.255
 rule 4 deny ip destination 172.31.7.0 0.0.0.255
 rule 5 deny ip destination 172.31.8.0 0.0.0.255
 rule 6 deny ip destination 172.31.9.0 0.0.0.255
 rule 7 deny ip destination 172.31.10.0 0.0.0.255
 rule 8 deny ip destination 172.31.11.0 0.0.0.255
 rule 9 deny ip destination 172.31.12.0 0.0.0.255
 rule 10 permit ip

traffic classifier 3001
 if-match acl 3001
traffic behavior 3001
 deny
traffic policy 3001
 classifier 3001 behavior 3001

vlan 100
 traffic-policy 3001 inbound
```

---

## 三、IP地址规划表

| 楼栋 | 业务类型 | VLAN | 网段 | 网关 |
|------|----------|------|------|------|
| 1#办公楼 | 有线 | 1 | 172.31.1.0/24 | 172.31.1.254 |
| 1#办公楼 | 无线Work | 2 | 172.31.2.0/24 | 172.31.2.254 |
| 1#办公楼 | AP管理 | 3 | 172.31.3.0/24 | 172.31.3.254 |
| 2#客房 | 有线 | 4 | 172.31.4.0/24 | 172.31.4.254 |
| 2#客房 | 无线Hotel | 50 | 172.31.5.0/24 | 172.31.5.254 |
| 2#客房 | AP管理 | 6 | 172.31.6.0/24 | 172.31.6.254 |
| 3#客房 | 有线 | 7 | 172.31.7.0/24 | 172.31.7.254 |
| 3#客房 | 无线Hotel | 50 | 172.31.8.0/24 | 172.31.8.254 |
| 3#客房 | AP管理 | 9 | 172.31.9.0/24 | 172.31.9.254 |
| 4#客房 | 有线 | 10 | 172.31.10.0/24 | 172.31.10.254 |
| 4#客房 | 无线Hotel | 50 | 172.31.11.0/24 | 172.31.11.254 |
| 4#客房 | AP管理 | 12 | 172.31.12.0/24 | 172.31.12.254 |
| 数据中心 | 服务器 | 100 | 172.31.100.0/24 | 172.31.100.254 |

---

## 四、验证命令

| 验证项 | 命令 | 预期结果 |
|--------|------|----------|
| AP上线 | `display ap all` | 状态normal |
| 路由表 | `display ip routing-table` | 包含所有业务网段 |
| OSPF邻居 | `display ospf peer` | 状态Full |
| DHCP分配 | `display dhcp server ip-in-use` | 有IP记录 |
| 办公访问服务器 | `ping 172.31.100.1` | 通 |
| 客房访问服务器 | `ping 172.31.100.1` | 不通 |

---

## 五、项目价值

本项目涵盖**VLAN、DHCP、OSPF、ACL、无线AC、防火墙**等企业级网络核心技术，通过配置清空重配和故障模拟，可系统提升网络规划、配置实施和故障排查能力，是网络工程师岗位的完整实战项目。
