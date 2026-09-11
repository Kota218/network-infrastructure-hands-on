# Network Design

## 1. 目的

Cisco Packet Tracerを使用し、小規模社内ネットワークを想定した冗長構成を設計・構築する。

本構成では、単一障害点を減らしつつ、VLAN単位で通信経路を分散し、障害発生時にも通信を継続できることを目的とする。

---

## 2. 要件

### ネットワーク要件

- 一般社員端末用ネットワーク
- サーバ用ネットワーク
- 開発端末用ネットワーク
- NW機器管理用ネットワーク
- VLAN間ルーティング
- デフォルトゲートウェイ冗長化
- L2経路冗長化
- 管理アクセス制御
- 通信制御
- 障害時の自動切替

### セキュリティ要件

- VLAN10からWEB01へのHTTP通信は許可
- VLAN10からWEB01へのHTTP以外の通信は拒否
- VLAN10からDB01への直接通信は拒否
- NW機器へのSSHはVLAN99からのみ許可
- VLAN10 / VLAN30からのSSHは禁止

---

## 3. 論理構成

```text
                     +----------------+
                     |     CORE01     |
                     | Catalyst 3650  |
                     +--------+-------+
                         |    |    |
                         |    |    |
             +-----------+    |    +-----------+
             |                |                |
             |                |                |
         +---+---+         CORE間          +---+---+
         | ASW01 |<----------------------->| CORE02|
         +---+---+                         +---+---+
             |                                 |
             |                                 |
         PC01/PC02                           ASW02
         WEB01                               / |  \
                                          DEV DB ADMIN
```

実際にはASW01 / ASW02は両COREへ接続し、L2冗長経路を持つ。

---

## 4. VLAN設計

| VLAN | Name | Purpose | Network |
|---:|---|---|---|
| 10 | PC | 一般社員端末 | 192.168.10.0/24 |
| 20 | SERVER | サーバ | 192.168.20.0/24 |
| 30 | DEVELOPER | 開発端末 | 192.168.30.0/24 |
| 99 | MANAGEMENT | NW機器管理 | 192.168.99.0/24 |

VLANごとに用途を分離し、ブロードキャストドメインと管理範囲を明確にする。

---

## 5. L3設計

### SVI

CORE01 / CORE02に各VLANのSVIを作成し、Inter-VLAN Routingを行う。

例:

```cisco
interface Vlan10
 ip address 192.168.10.2 255.255.255.0
```

CORE側では `ip routing` を有効化する。

### デフォルトゲートウェイ

端末のデフォルトゲートウェイには、各VLANのHSRP Virtual IPを設定する。

| VLAN | Virtual Gateway |
|---:|---|
| 10 | 192.168.10.1 |
| 20 | 192.168.20.1 |
| 30 | 192.168.30.1 |
| 99 | 192.168.99.1 |

---

## 6. HSRP設計

CORE01 / CORE02でHSRPを構成し、デフォルトゲートウェイを冗長化する。

### Active配置

| VLAN | Active | Standby |
|---:|---|---|
| 10 | CORE01 | CORE02 |
| 20 | CORE02 | CORE01 |
| 30 | CORE01 | CORE02 |
| 99 | CORE02 | CORE01 |

VLAN単位でActiveを分散し、通常時から両COREを利用する。

### Priority

- Active側: 105
- Standby側: 100
- `preempt` 有効

`preempt` を有効化することで、障害復旧後に設計上の優先COREへActiveを戻す。

---

## 7. STP設計

L2ループを防止しつつ、ASWからCOREへの冗長経路を確保する。

### Root配置

| VLAN | Root Primary | Root Secondary |
|---:|---|---|
| 10 | CORE01 | CORE02 |
| 20 | CORE02 | CORE01 |
| 30 | CORE01 | CORE02 |
| 99 | CORE02 | CORE01 |

HSRP ActiveとSTP Rootを一致させる。

これにより、例えばVLAN10では以下のように通信経路を揃える。

```text
PC01
  |
ASW01
  |
CORE01
  |
HSRP Virtual GW
```

HSRP ActiveとSTP Rootが異なると、不要なL2迂回が発生する可能性があるため、役割を揃える設計とした。

---

## 8. Trunk / Access設計

### CORE間

CORE01 - CORE02間はtrunkとし、全VLANを許可する。

```text
Allowed VLAN:
10,20,30,99
```

### CORE - ASW間

ASWごとに必要なVLANのみ許可する。

ASW01:

```text
10,20,99
```

ASW02:

```text
20,30,99
```

不要なVLANをtrunkへ流さないことで、構成を明確にする。

### Access Port

端末接続ポートはaccessとする。

- PC01 / PC02 → VLAN10
- WEB01 / DB01 → VLAN20
- DEV01 → VLAN30
- ADMIN01 → VLAN99

---

## 9. ASW管理設計

ASWはL2スイッチとして使用するため、ユーザVLAN用SVIは作成しない。

管理用としてVLAN99のみSVIを設定する。

```text
ASW01: 192.168.99.4/24
ASW02: 192.168.99.5/24
```

Default Gateway:

```text
192.168.99.1
```

---

## 10. ACL設計

### VLAN10-IN

拡張ACLをVlan10のinboundへ適用する。

```text
VLAN10
  |
  | inbound
  v
CORE SVI
  |
  +-- WEB01 HTTP -> Permit
  +-- WEB01 Other -> Deny
  +-- DB01        -> Deny
  +-- Others      -> Permit
```

ACLはCORE01 / CORE02の両方へ同じ内容を設定する。

理由:

CORE01障害時にCORE02がHSRP Activeへ切り替わっても、同じセキュリティポリシーを維持するため。

---

## 11. SSH管理設計

NW機器へのリモート管理はSSHのみとする。

```cisco
transport input ssh
```

標準ACLをVTYへ適用し、接続元をVLAN99に限定する。

```cisco
ip access-list standard SSH-MGMT
 permit 192.168.99.0 0.0.0.255
```

```cisco
line vty 0 4
 access-class SSH-MGMT in
```

---

## 12. PortFast / BPDU Guard

PCやServerなどの端末接続ポートにはPortFastを設定する。

```cisco
spanning-tree portfast
```

さらにBPDU Guardを設定し、端末ポートに誤ってSWが接続された場合に保護する。

```cisco
spanning-tree bpduguard enable
```

---

## 13. 障害時の動作設計

### CORE01 - ASW01リンク障害

正常時:

```text
ASW01 Gi0/1 -> CORE01 -> Root FWD
ASW01 Gi0/2 -> CORE02 -> Altn BLK
```

障害後:

```text
ASW01 Gi0/2 -> CORE02 -> Root FWD
```

STP収束後に通信が復旧する。

### CORE01装置障害

VLAN10 / VLAN30:

```text
HSRP Active
CORE01 -> CORE02
```

STP:

```text
Root
CORE01 -> CORE02
```

通信は一時的に停止するが、HSRP / STP収束後にCORE02経由で復旧する。

### CORE01復旧

`preempt` によりVLAN10 / VLAN30のHSRP ActiveはCORE01へ戻る。

STPも設計上のRootへ復帰する。

---

## 14. 設計上のポイント

### 1. HSRPとSTPの役割を分離して考える

- HSRP: L3デフォルトゲートウェイ冗長化
- STP: L2ループ防止・経路冗長化

### 2. Active / Rootを揃える

HSRP ActiveとSTP Rootを一致させ、通常時の通信経路を最適化する。

### 3. 冗長系にも同じセキュリティ設定を入れる

ACLやSSH制御をActive機だけに設定すると、障害切替時にポリシーが変わるため、両COREへ設定する。

### 4. 障害試験まで行う

設定投入後に以下を確認する。

- HSRP role
- STP root / root port
- ACL hit count
- SSH access control
- Link failover
- Core failover
- Failback

---

## 15. 今後の拡張

今後追加する場合は、以下を検討する。

- Router / Internet側接続
- OSPF
- NAT
- DHCP
- NTP
- Syslog
- SNMP
- Config Backup
- Port Security
- AAA
- RADIUS / TACACS+
