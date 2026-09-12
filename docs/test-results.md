# Test Results

## 1. 概要

Cisco Packet Tracerで構築した小規模社内ネットワークについて、正常系・通信制御・冗長化・障害復旧の各試験を実施した結果をまとめます。

---

## 2. 正常系疎通試験

| No. | Source | Destination | Test | Expected | Result |
|---:|---|---|---|---|---|
| 1 | PC01 | PC02 | ICMP | Success | OK |
| 2 | PC01 | WEB01 | ICMP | Success（ACL投入前） | OK |
| 3 | PC01 | DB01 | ICMP | Success（ACL投入前） | OK |
| 4 | PC01 | DEV01 | ICMP | Success | OK |
| 5 | PC01 | ADMIN01 | ICMP | Success | OK |
| 6 | PC01 | VLAN10 Virtual GW | ICMP | Success | OK |
| 7 | PC01 | VLAN20 Virtual GW | ICMP | Success | OK |
| 8 | PC01 | VLAN30 Virtual GW | ICMP | Success | OK |
| 9 | PC01 | VLAN99 Virtual GW | ICMP | Success | OK |

---

## 3. ACL試験

### VLAN10-IN

要件:

- VLAN10 → WEB01 HTTP: Permit
- VLAN10 → WEB01 その他: Deny
- VLAN10 → DB01: Deny
- VLAN10 → その他: Permit

| No. | Source | Destination | Protocol | Expected | Result |
|---:|---|---|---|---|---|
| 10 | PC01 | WEB01 | HTTP / TCP 80 | Permit | OK |
| 11 | PC01 | WEB01 | ICMP | Deny | OK |
| 12 | PC01 | DB01 | ICMP | Deny | OK |
| 13 | PC01 | DEV01 | ICMP | Permit | OK |
| 14 | PC01 | ADMIN01 | ICMP | Permit | OK |

確認コマンド:

```cisco
show access-lists
show ip interface vlan 10
```

確認結果:

- HTTPはWeb Browserから正常表示
- WEB01 / DB01へのICMPは拒否
- ACL hit countが試験通信に応じて増加することを確認

### Evidence

HTTP許可:

![ACL HTTP Permit](../evidence/acl-test.png)

WEB01 / DB01へのICMP拒否:

![ACL Deny Test](../evidence/acl-deny-test.png)

ACL hit count:

![ACL Hit Count](../evidence/acl-hitcount.png)

---

## 4. SSH管理アクセス試験

要件:

- VLAN99からNW機器へのSSH: Permit
- VLAN10 / VLAN30からNW機器へのSSH: Deny

| No. | Source | Destination | Expected | Result |
|---:|---|---|---|---|
| 15 | ADMIN01 (VLAN99) | CORE01 | SSH Success | OK |
| 16 | ADMIN01 (VLAN99) | CORE02 | SSH Success | OK |
| 17 | ADMIN01 (VLAN99) | ASW01 | SSH Success | OK |
| 18 | ADMIN01 (VLAN99) | ASW02 | SSH Success | OK |
| 19 | PC01 (VLAN10) | CORE01 / ASW01 | SSH Deny | OK |
| 20 | DEV01 (VLAN30) | CORE01 | SSH Deny | OK |

確認コマンド:

```cisco
show ip ssh
show access-lists SSH-MGMT
```

確認結果:

- ADMIN01から4台すべてへSSHログイン成功
- PC01からCORE01 / ASW01へのSSH接続を拒否
- DEV01からCORE01へのSSH接続を拒否
- `SSH-MGMT` のpermit側hit count増加を確認

---

## 5. STPリンク障害試験（VLAN10）

### 対象

- VLAN: VLAN10
- Link: CORE01 `Gi1/0/2` - ASW01 `Gi0/1`

### 正常時

```text
ASW01 Gi0/1 -> Root FWD
ASW01 Gi0/2 -> Altn BLK
Root Cost   -> 4
```

### 障害操作

CORE01:

```cisco
configure terminal
interface GigabitEthernet1/0/2
 shutdown
end
```

### 障害後

```text
ASW01 Gi0/2 -> Root FWD
Root Cost   -> 8
```

PC01 → DEV01:

- 障害直後: 一時的に通信断
- STP収束後: 通信復旧

### 復旧操作

```cisco
configure terminal
interface GigabitEthernet1/0/2
 no shutdown
end
```

### 復旧後

```text
ASW01 Gi0/1 -> Root FWD
ASW01 Gi0/2 -> Altn BLK
Root Cost   -> 4
```

結果: **OK**

---

## 6. CORE01装置障害試験

### 障害操作

Packet Tracer上でCORE01の電源をOFF。

### 期待結果

- VLAN10 HSRP Active: CORE01 → CORE02
- VLAN30 HSRP Active: CORE01 → CORE02
- VLAN10 STP Root: CORE01 → CORE02
- VLAN30 STP Root: CORE01 → CORE02
- 収束後にVLAN間通信が復旧

### 実結果

CORE02:

```text
VLAN10 State is Active
VLAN30 State is Active
```

STP:

```text
VLAN10: This bridge is the root
VLAN30: This bridge is the root
```

PC01 → DEV01:

- 障害直後: 100% loss
- 収束後: 4/4 reply

結果: **OK**

---

## 7. フェイルバック試験

### 復旧操作

CORE01の電源をON。

### 期待結果

HSRP:

- VLAN10 → CORE01 Activeへ復帰
- VLAN20 → CORE02 Activeを維持
- VLAN30 → CORE01 Activeへ復帰
- VLAN99 → CORE02 Activeを維持

STP:

- VLAN10 / VLAN30 → CORE01 Rootへ復帰
- VLAN20 / VLAN99 → CORE02 Rootを維持

### 実結果

CORE01 `show standby`:

```text
VLAN10 State is Active
VLAN20 State is Standby
VLAN30 State is Active
VLAN99 State is Standby
```

CORE01 `show spanning-tree vlan 10`:

```text
This bridge is the root
```

結果: **OK**

---

## 8. 総合結果

| Category | Result |
|---|---|
| VLAN / Trunk / Access | OK |
| Inter-VLAN Routing | OK |
| HSRP | OK |
| STP | OK |
| ACL | OK |
| SSH Access Control | OK |
| Link Failover | OK |
| Core Failover | OK |
| Failback | OK |

---

## 9. Evidence

```text
evidence/
├── acl-test.png
├── acl-deny-test.png
├── acl-hitcount.png
├── stp-failover.png
├── hsrp-failover.png
├── failback.png
├── ssh-vlan99-success.png
├── ssh-vlan10-deny.png
├── ssh-vlan30-deny.png
└── ssh-acl-hitcount.png
```

GitHubへ公開する際は、個人情報・認証情報・実環境情報が写り込んでいないことを確認する。
