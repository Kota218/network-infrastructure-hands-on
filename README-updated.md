# Network Infrastructure Hands-on

Cisco Packet Tracerを使用し、**冗長化・VLAN間ルーティング・ACL・SSH管理・障害試験**まで含めた小規模社内ネットワークを設計・構築したハンズオンです。

単に通信を通すだけではなく、**要件整理 → 設計 → 構築 → 正常性確認 → 障害試験 → 復旧確認**までを一連の流れとして実施しています。

---

## Documents

- [Network Design](docs/network-design.md)
- [Parameter Sheet](docs/parameter-sheet.md)
- [Test Results](docs/test-results.md)

---

## 1. 構成概要

### ネットワーク機器

| Hostname | Device | Role |
|---|---|---|
| CORE01 | Catalyst 3650 | L3 Core Switch |
| CORE02 | Catalyst 3650 | L3 Core Switch |
| ASW01 | Catalyst 2960 | Access Switch |
| ASW02 | Catalyst 2960 | Access Switch |

### VLAN

| VLAN | Name | Purpose | Network |
|---:|---|---|---|
| 10 | PC | 一般社員端末 | 192.168.10.0/24 |
| 20 | SERVER | サーバ | 192.168.20.0/24 |
| 30 | DEVELOPER | 開発端末 | 192.168.30.0/24 |
| 99 | MANAGEMENT | NW機器管理 | 192.168.99.0/24 |

---

## 2. 実装内容

- VLAN / Access Port / Trunk
- SVIによるInter-VLAN Routing
- HSRPによるデフォルトゲートウェイ冗長化
- HSRP Priority / Preempt
- STPによるL2ループ防止・冗長経路制御
- HSRP ActiveとSTP Rootの役割分散
- Extended ACLによる通信制御
- Standard ACL + VTY `access-class` によるSSH接続元制限
- PortFast / BPDU Guard
- リンク障害試験
- COREスイッチ障害試験
- フェイルオーバー / フェイルバック確認
- ACL hit count確認

---

## 3. 冗長化設計

通常時は、HSRP ActiveとSTP Rootを同じCOREに揃えています。

| VLAN | HSRP Active | STP Root |
|---:|---|---|
| 10 | CORE01 | CORE01 |
| 20 | CORE02 | CORE02 |
| 30 | CORE01 | CORE01 |
| 99 | CORE02 | CORE02 |

---

## 4. 通信制御

VLAN10からの通信は以下の要件で制御しています。

- WEB01へのHTTP通信：許可
- WEB01へのHTTP以外：拒否
- DB01への直接通信：拒否
- その他の通信：許可

```cisco
ip access-list extended VLAN10-IN
 permit tcp 192.168.10.0 0.0.0.255 host 192.168.20.11 eq 80
 deny ip 192.168.10.0 0.0.0.255 host 192.168.20.11
 deny ip 192.168.10.0 0.0.0.255 host 192.168.20.12
 permit ip 192.168.10.0 0.0.0.255 any
```

![ACL Test](evidence/acl-test.png)

---

## 5. SSH管理

CORE01 / CORE02 / ASW01 / ASW02へのSSH接続は、管理VLANであるVLAN99からのみ許可しています。

```cisco
ip access-list standard SSH-MGMT
 permit 192.168.99.0 0.0.0.255
```

```cisco
line vty 0 4
 login local
 transport input ssh
 access-class SSH-MGMT in
```

確認結果:

- ADMIN01 (VLAN99) → 4台すべてSSH成功
- PC01 (VLAN10) → SSH拒否
- DEV01 (VLAN30) → SSH拒否

### VLAN99からのSSH成功

![SSH VLAN99 Success](evidence/ssh-vlan99-success.png)

### VLAN10からのSSH拒否

![SSH VLAN10 Deny](evidence/ssh-vlan10-deny.png)

### VLAN30からのSSH拒否

![SSH VLAN30 Deny](evidence/ssh-vlan30-deny.png)

### SSH管理ACL hit count

![SSH ACL Hit Count](evidence/ssh-acl-hitcount.png)

---

## 6. 障害試験

### STPリンク障害

CORE01 - ASW01間リンクを停止し、ASW01のRoot PortがCORE02側へ切り替わることを確認しました。

![STP Failover](evidence/stp-failover.png)

### HSRP / CORE障害

CORE01を停止し、VLAN10 / VLAN30のHSRP ActiveがCORE02へ切り替わることを確認しました。

![HSRP Failover](evidence/hsrp-failover.png)

### フェイルバック

CORE01復旧後、HSRP PreemptおよびSTPにより元の設計状態へ戻ることを確認しました。

![Failback](evidence/failback.png)

---

## 7. Config Files

- [CORE01](configs/CORE01.txt)
- [CORE02](configs/CORE02.txt)
- [ASW01](configs/ASW01.txt)
- [ASW02](configs/ASW02.txt)

---

## 8. 主な確認コマンド

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
show ip ssh
```

---

## 9. Repository Structure

```text
network-infrastructure-hands-on/
├── README.md
├── docs/
│   ├── parameter-sheet.md
│   ├── test-results.md
│   └── network-design.md
├── configs/
│   ├── CORE01.txt
│   ├── CORE02.txt
│   ├── ASW01.txt
│   └── ASW02.txt
└── evidence/
    ├── acl-test.png
    ├── stp-failover.png
    ├── hsrp-failover.png
    ├── failback.png
    ├── ssh-vlan99-success.png
    ├── ssh-vlan10-deny.png
    ├── ssh-vlan30-deny.png
    └── ssh-acl-hitcount.png
```

---

## 10. 学習・検証ポイント

- 要件からVLAN / IP / ポート構成を設計
- HSRPとSTPの役割を分離して理解
- HSRP ActiveとSTP Rootを揃えた経路設計
- ACLの評価順序と暗黙のdenyを確認
- SSH管理元をVLAN99に限定
- 端末ポートにPortFast / BPDU Guardを設定
- リンク障害・CORE障害・復旧まで一連で検証
- `show` コマンドとACL hit countで設定結果を確認

---

## Notes

このリポジトリは学習用の検証環境です。

公開リポジトリには、実運用で使用するパスワード、秘密鍵、顧客情報、実環境のConfigなどは保存しません。
