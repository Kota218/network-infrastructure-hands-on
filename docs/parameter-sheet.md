# Network Hands-on Parameter Sheet

## 1. 概要

Packet Tracerを使用した小規模社内ネットワーク構築ハンズオンのパラメータシートです。

- COREスイッチ2台による冗長構成
- HSRPによるデフォルトゲートウェイ冗長化
- STPによるL2ループ防止・経路冗長化
- VLAN間ルーティング
- ACLによる通信制御
- VLAN99からのみSSH管理を許可
- CORE障害・リンク障害時のフェイルオーバー確認

---

## 2. 機器一覧

| Hostname | 機器 | 役割 |
|---|---|---|
| CORE01 | Catalyst 3650 | L3 Core Switch / HSRP / STP |
| CORE02 | Catalyst 3650 | L3 Core Switch / HSRP / STP |
| ASW01 | Catalyst 2960 | Access Switch |
| ASW02 | Catalyst 2960 | Access Switch |
| PC01 | PC | 一般社員端末 |
| PC02 | PC | 一般社員端末 |
| WEB01 | Server | Web Server |
| DB01 | Server | Database Server |
| DEV01 | PC | 開発端末 |
| ADMIN01 | PC | 管理端末 |

---

## 3. VLAN設計

| VLAN ID | VLAN名 | 用途 | Network |
|---:|---|---|---|
| 10 | PC | 一般社員端末 | 192.168.10.0/24 |
| 20 | SERVER | サーバ | 192.168.20.0/24 |
| 30 | DEVELOPER | 開発端末 | 192.168.30.0/24 |
| 99 | MANAGEMENT | NW機器管理 | 192.168.99.0/24 |

---

## 4. HSRP / SVI設計

### CORE01

| VLAN | SVI IP | HSRP Virtual IP | Group | Priority | HSRP Role | Preempt |
|---:|---|---|---:|---:|---|---|
| 10 | 192.168.10.2/24 | 192.168.10.1 | 10 | 105 | Active | 有効 |
| 20 | 192.168.20.2/24 | 192.168.20.1 | 20 | 100 | Standby | 有効 |
| 30 | 192.168.30.2/24 | 192.168.30.1 | 30 | 105 | Active | 有効 |
| 99 | 192.168.99.2/24 | 192.168.99.1 | 99 | 100 | Standby | 有効 |

### CORE02

| VLAN | SVI IP | HSRP Virtual IP | Group | Priority | HSRP Role | Preempt |
|---:|---|---|---:|---:|---|---|
| 10 | 192.168.10.3/24 | 192.168.10.1 | 10 | 100 | Standby | 有効 |
| 20 | 192.168.20.3/24 | 192.168.20.1 | 20 | 105 | Active | 有効 |
| 30 | 192.168.30.3/24 | 192.168.30.1 | 30 | 100 | Standby | 有効 |
| 99 | 192.168.99.3/24 | 192.168.99.1 | 99 | 105 | Active | 有効 |

---

## 5. STP設計

HSRP ActiveとSTP Rootを一致させ、通常時のL2経路とデフォルトゲートウェイを揃える。

| VLAN | STP Root Primary | STP Root Secondary |
|---:|---|---|
| 10 | CORE01 | CORE02 |
| 20 | CORE02 | CORE01 |
| 30 | CORE01 | CORE02 |
| 99 | CORE02 | CORE01 |

---

## 6. CORE01 ポート設計

| Interface | 接続先 | Mode | Allowed VLAN | IP |
|---|---|---|---|---|
| Gi1/0/1 | CORE02 Gi1/0/1 | trunk | 10,20,30,99 | - |
| Gi1/0/2 | ASW01 Gi0/1 | trunk | 10,20,99 | - |
| Gi1/0/3 | ASW02 Gi0/1 | trunk | 20,30,99 | - |
| Vlan10 | - | SVI | VLAN10 | 192.168.10.2/24 |
| Vlan20 | - | SVI | VLAN20 | 192.168.20.2/24 |
| Vlan30 | - | SVI | VLAN30 | 192.168.30.2/24 |
| Vlan99 | - | SVI | VLAN99 | 192.168.99.2/24 |

---

## 7. CORE02 ポート設計

| Interface | 接続先 | Mode | Allowed VLAN | IP |
|---|---|---|---|---|
| Gi1/0/1 | CORE01 Gi1/0/1 | trunk | 10,20,30,99 | - |
| Gi1/0/2 | ASW01 Gi0/2 | trunk | 10,20,99 | - |
| Gi1/0/3 | ASW02 Gi0/2 | trunk | 20,30,99 | - |
| Vlan10 | - | SVI | VLAN10 | 192.168.10.3/24 |
| Vlan20 | - | SVI | VLAN20 | 192.168.20.3/24 |
| Vlan30 | - | SVI | VLAN30 | 192.168.30.3/24 |
| Vlan99 | - | SVI | VLAN99 | 192.168.99.3/24 |

---

## 8. ASW01 ポート設計

| Interface | 接続先 | Mode | VLAN / Allowed VLAN | 備考 |
|---|---|---|---|---|
| Gi0/1 | CORE01 Gi1/0/2 | trunk | 10,20,99 | Uplink |
| Gi0/2 | CORE02 Gi1/0/2 | trunk | 10,20,99 | Uplink |
| Fa0/1 | PC01 | access | VLAN10 | 社員端末 |
| Fa0/2 | PC02 | access | VLAN10 | 社員端末 |
| Fa0/3 | WEB01 | access | VLAN20 | Web Server |
| Vlan99 | - | Management SVI | VLAN99 | 192.168.99.4/24 |

Default Gateway: `192.168.99.1`

---

## 9. ASW02 ポート設計

| Interface | 接続先 | Mode | VLAN / Allowed VLAN | 備考 |
|---|---|---|---|---|
| Gi0/1 | CORE01 Gi1/0/3 | trunk | 20,30,99 | Uplink |
| Gi0/2 | CORE02 Gi1/0/3 | trunk | 20,30,99 | Uplink |
| Fa0/1 | DEV01 | access | VLAN30 | 開発端末 |
| Fa0/2 | DB01 | access | VLAN20 | DB Server |
| Fa0/3 | ADMIN01 | access | VLAN99 | 管理端末 |
| Vlan99 | - | Management SVI | VLAN99 | 192.168.99.5/24 |

Default Gateway: `192.168.99.1`

---

## 10. 端末IPアドレス

| Hostname | VLAN | IP Address | Subnet Mask | Default Gateway |
|---|---:|---|---|---|
| PC01 | 10 | 192.168.10.11 | 255.255.255.0 | 192.168.10.1 |
| PC02 | 10 | 192.168.10.12 | 255.255.255.0 | 192.168.10.1 |
| WEB01 | 20 | 192.168.20.11 | 255.255.255.0 | 192.168.20.1 |
| DB01 | 20 | 192.168.20.12 | 255.255.255.0 | 192.168.20.1 |
| DEV01 | 30 | 192.168.30.10 | 255.255.255.0 | 192.168.30.1 |
| ADMIN01 | 99 | 192.168.99.10 | 255.255.255.0 | 192.168.99.1 |

---

## 11. ACL設計

### VLAN10-IN

適用先:

- CORE01 `interface Vlan10`
- CORE02 `interface Vlan10`
- Direction: `in`

目的:

- VLAN10からWEB01へのHTTP通信のみ許可
- VLAN10からWEB01へのその他通信を拒否
- VLAN10からDB01への直接通信を拒否
- その他の通信は許可

| Seq | Action | Protocol | Source | Destination | Port |
|---:|---|---|---|---|---|
| 10 | permit | tcp | 192.168.10.0/24 | 192.168.20.11 | 80 |
| 20 | deny | ip | 192.168.10.0/24 | 192.168.20.11 | any |
| 30 | deny | ip | 192.168.10.0/24 | 192.168.20.12 | any |
| 40 | permit | ip | 192.168.10.0/24 | any | any |

Config:

```cisco
ip access-list extended VLAN10-IN
 permit tcp 192.168.10.0 0.0.0.255 host 192.168.20.11 eq 80
 deny ip 192.168.10.0 0.0.0.255 host 192.168.20.11
 deny ip 192.168.10.0 0.0.0.255 host 192.168.20.12
 permit ip 192.168.10.0 0.0.0.255 any
```

---

## 12. SSH管理設計

NW機器へのSSH接続はVLAN99からのみ許可する。

### SSH-MGMT

```cisco
ip access-list standard SSH-MGMT
 permit 192.168.99.0 0.0.0.255
```

VTY:

```cisco
line vty 0 4
 login local
 transport input ssh
 access-class SSH-MGMT in
```

通信要件:

| Source | NW機器へのSSH |
|---|---|
| VLAN99 | Permit |
| VLAN10 | Deny |
| VLAN30 | Deny |

※公開リポジトリには実運用パスワード・秘密鍵を記載しない。

---

## 13. PortFast / BPDU Guard

ASW01 / ASW02の端末接続ポートに設定。

```cisco
interface range FastEthernet0/1-3
 spanning-tree portfast
 spanning-tree bpduguard enable
```

---

## 14. 正常時の期待状態

### HSRP

| VLAN | Active | Standby |
|---:|---|---|
| 10 | CORE01 | CORE02 |
| 20 | CORE02 | CORE01 |
| 30 | CORE01 | CORE02 |
| 99 | CORE02 | CORE01 |

### STP

| VLAN | Root Bridge |
|---:|---|
| 10 | CORE01 |
| 20 | CORE02 |
| 30 | CORE01 |
| 99 | CORE02 |

---

## 15. 障害試験結果

| No. | 試験 | 期待結果 | 結果 |
|---:|---|---|---|
| 1 | CORE01-ASW01リンク停止 | STPがCORE02側へ切替 | OK |
| 2 | CORE01-ASW01リンク復旧 | STPがCORE01側へ復帰 | OK |
| 3 | CORE01電源停止 | VLAN10/30のHSRP ActiveがCORE02へ切替 | OK |
| 4 | CORE01電源停止 | VLAN10/30のSTP RootがCORE02へ切替 | OK |
| 5 | CORE01停止後のVLAN間通信 | 収束後に通信復旧 | OK |
| 6 | CORE01復旧 | HSRP PreemptによりVLAN10/30がCORE01へ復帰 | OK |
| 7 | CORE01復旧 | STP Rootが設計状態へ復帰 | OK |
| 8 | VLAN10 → WEB01 HTTP | Permit | OK |
| 9 | VLAN10 → WEB01 ICMP | Deny | OK |
| 10 | VLAN10 → DB01 | Deny | OK |
| 11 | VLAN99 → CORE01 / CORE02 / ASW01 / ASW02 SSH | Permit | OK |
| 12 | VLAN10 → CORE01 / ASW01 SSH | Deny | OK |
| 13 | VLAN30 → CORE01 SSH | Deny | OK |

---

## 16. 確認コマンド

```cisco
show vlan brief
show interfaces trunk
show ip interface brief
show standby
show spanning-tree vlan 10
show spanning-tree vlan 20
show spanning-tree vlan 30
show spanning-tree vlan 99
show access-lists
show ip interface vlan 10
```

---

## 17. 備考

- CORE間・CORE-ASW間は802.1Q trunkを使用する。
- CORE側物理ポートにはIPアドレスを設定せず、SVIでL3処理を行う。
- ASWはL2スイッチとして使用し、管理用VLAN99のみSVIを設定する。
- HSRP ActiveとSTP Rootを同一COREに揃え、不要なL2迂回を抑える。
- CORE01/CORE02の両方に同一のVLAN10 ACLを設定し、フェイルオーバー後も同じ通信制御を維持する。
- SSH管理ACLも冗長系を含む各NW機器へ展開する。
- 公開GitHubには実運用の認証情報・秘密情報を保存しない。
