# 3.3 IP サービス

使用デバイス: **R1 / R2 / R3 / R4** (PC1)

---

## 3.3.a NTP や PTP などのネットワーク時間プロトコル設定の理解 { #33a-ntp-ptp }

!!! abstract "所要 1 分 / 目標"
    NTP の階層 (ストラタム) と認証付き設定を行い、PTP との違いと基本用語を説明できる。

!!! warning "最初に設定して、最後に確認します"
    NTP の同期には数分かかります。ここで設定を投入したら次へ進み、
    [まとめ](99-summary.md) の時点で同期状態を確認してください。

### ハンズオン (NTP)

R1 を NTP サーバー (ストラタム 3) にし、R2 / R3 をクライアントにします。

=== "R1 (サーバー)"

    ```text
    clock timezone JST 9 0
    ntp authentication-key 1 md5 CCNP-NTP-KEY
    ntp trusted-key 1
    ntp source Loopback0
    ntp master 3
    ```

=== "R2 (クライアント)"

    ```text
    clock timezone JST 9 0
    ntp authentication-key 1 md5 CCNP-NTP-KEY
    ntp authenticate
    ntp trusted-key 1
    ntp server 1.1.1.1 key 1 source Loopback0
    ```

=== "R3 (クライアント)"

    ```text
    clock timezone JST 9 0
    ntp authentication-key 1 md5 CCNP-NTP-KEY
    ntp authenticate
    ntp trusted-key 1
    ntp server 1.1.1.1 key 1 source Loopback0
    ```

```text
R2# show ntp status
R2# show ntp associations
R1# show clock detail
```

期待結果 (数分後): `Clock is synchronized, stratum 4, reference is 1.1.1.1` /
associations で `*` (sys.peer) が表示される。

### NTP の要点

| 項目 | 内容 |
|------|------|
| ストラタム | 0 = 基準時計 (GPS / 原子時計)。1〜15 = 階層。**16 = 非同期** |
| `ntp server` | 上位サーバーに同期するクライアント |
| `ntp peer` | 対等な相手と相互に同期 (対称アクティブ) |
| `ntp master <stratum>` | 外部ソースなしで自身を時刻源にする (**ラボ向け**) |
| 認証 | `ntp authentication-key` / `ntp authenticate` / `ntp trusted-key` |
| アクセス制御 | `ntp access-group` |
| 通信 | UDP 123 |

!!! tip "試験ポイント"
    - クライアント側で **`ntp authenticate` と `ntp trusted-key` の両方**が必要。
    - 時刻差が大きいと (約 1000 秒超) 同期できない。事前に `clock set` で合わせる。
    - ログや証明書、Kerberos、ログ相関のために時刻同期は必須。

### PTP (Precision Time Protocol, IEEE 1588)

| 比較項目 | NTP | PTP (IEEE 1588v2) |
|----------|-----|-------------------|
| 精度 | ミリ秒〜 | **サブマイクロ秒〜ナノ秒** |
| タイムスタンプ | ソフトウェア | **ハードウェア** (NIC / ASIC) |
| 通信 | UDP 123 (ユニキャスト) | UDP 319 (イベント) / 320 (一般)。マルチキャスト 224.0.1.129、または L2 (Ethernet) |
| 用途 | 一般的な IT システム | 金融取引、放送 (SMPTE 2059)、通信 (4G/5G の同期)、産業用 |
| 階層 | ストラタム | **グランドマスター → 境界クロック → スレーブ** |

**PTP のクロックタイプ**

| タイプ | 説明 |
|--------|------|
| **Ordinary Clock (OC)** | 単一ポートのクロック。**Grandmaster (GM)** またはスレーブ |
| **Boundary Clock (BC)** | 複数ポート。上流に対してはスレーブ、下流に対してはマスター。**ネットワーク遅延の累積を断ち切る** |
| **Transparent Clock (TC)** | 通過するメッセージに滞留時間を補正値として加算 (E2E / P2P) |

**主要メッセージ**: `Announce` (BMCA 用) / `Sync` / `Follow_Up` / `Delay_Req` / `Delay_Resp`

**BMCA (Best Master Clock Algorithm)** の比較順:
`Priority1` → `Clock Class` → `Clock Accuracy` → `Offset Scaled Log Variance` → `Priority2` → `Clock Identity`

**設定例 (Catalyst 9000 系の例。プラットフォームによりコマンドは異なります。ラボには適用しません)**

```text
ptp mode boundary delay-req
ptp priority1 100
ptp priority2 100
ptp domain 0
```

```text
show ptp clock
show ptp brief
show ptp port
```

---

## 3.3.b NAT/PAT の設定 { #33b-natpat }

!!! abstract "所要 3 分 / 目標"
    静的 NAT・動的 NAT・PAT の違いを理解し、PAT を設定して変換テーブルを確認できる。

### 用語 (4 種類のアドレス)

| 用語 | 意味 |
|------|------|
| **Inside Local** | 内部ホストの実アドレス (例: 192.168.10.100) |
| **Inside Global** | 外部から見た内部ホストのアドレス (例: 10.0.24.1:ポート) |
| **Outside Local** | 内部から見た外部ホストのアドレス |
| **Outside Global** | 外部ホストの実アドレス (例: 203.0.113.1) |

| 種類 | 変換 | コマンドの要点 |
|------|------|----------------|
| **静的 NAT** | 1 対 1 (固定) | `ip nat inside source static <local> <global>` |
| **動的 NAT** | プールからの 1 対 1 | `ip nat pool` + `ip nat inside source list <ACL> pool <名前>` |
| **PAT (NAT オーバーロード)** | 多対 1 (ポート番号で識別) | `ip nat inside source list <ACL> interface <IF> overload` |

### ハンズオン (R2)

R2 を境界ルーターとし、**LAN (192.168.10.0/24) を Gi0/3 の IP アドレスへ PAT** します。

```text
access-list 10 permit 192.168.10.0 0.0.0.255
!
interface GigabitEthernet0/1
 ip nat inside
interface GigabitEthernet0/3
 ip nat outside
!
ip nat inside source list 10 interface GigabitEthernet0/3 overload
```

**参考: 静的 NAT (サーバー公開) と動的 NAT** — 公開アドレス 198.51.100.0/24 は 3.2.c で BGP 広告済みです。

```text
! 静的 NAT: 内部サーバー 192.168.10.200 を 198.51.100.200 として公開
ip nat inside source static 192.168.10.200 198.51.100.200
!
! 動的 NAT: プールを利用 (PAT と同じ ACL を使う場合は、どちらか一方のみ有効にする)
ip nat pool PUBLIC 198.51.100.10 198.51.100.20 netmask 255.255.255.0
! ip nat inside source list 10 pool PUBLIC
```

### 確認

```text
PC1# ping 203.0.113.1
PC1# ping 203.0.113.1 repeat 10

R2# show ip nat translations
R2# show ip nat statistics
```

期待結果の例:

```text
Pro Inside global      Inside local       Outside local      Outside global
icmp 10.0.24.1:1       192.168.10.100:1   203.0.113.1:1      203.0.113.1:1
```

- **Inside global** が R2 の Gi0/3 (10.0.24.1) になり、ID (ポート) で区別されていれば PAT 成功。
- `show ip nat statistics` で `Hits` / `Misses` が増えること、`Inside interfaces` / `Outside interfaces` が正しいことを確認。

### 壊して直す

inside / outside を入れ替えて、変換されなくなることを確認します。

```text
R2(config)# interface GigabitEthernet0/1
R2(config-if)# no ip nat inside
R2(config-if)# ip nat outside
PC1# ping 203.0.113.1
R2# show ip nat statistics
```

- 復旧: `no ip nat outside` → `ip nat inside`。
- 既存の変換が残る場合は `clear ip nat translation *`。

**NAT トラブルシュートのチェックリスト**

1. `ip nat inside` / `ip nat outside` が**正しいインターフェイス**に付いているか
2. 変換対象 ACL が送信元をカバーしているか (`show access-lists`)
3. 外部から**戻りの経路**があるか (PAT でインターフェイスアドレスを使う場合は直結で OK)
4. プールのアドレスがインターフェイスや他のプールと重複していないか
5. `debug ip nat` で変換の様子を確認 (本番では注意)

---

## 3.3.c HSRP、VRRP などのファーストホップ冗長プロトコルの設定 { #33c-hsrp-vrrp }

!!! abstract "所要 3 分 / 目標"
    HSRP でゲートウェイ冗長を構成し、アクティブ/スタンバイの切り替えを確認する。
    VRRP との違いを説明できる。

### 比較

| 項目 | **HSRP** | **VRRP** | GLBP |
|------|----------|----------|------|
| 規格 | Cisco 独自 | IETF (RFC 5798) | Cisco 独自 |
| 役割 | Active / Standby | **Master** / Backup | AVG / AVF (**負荷分散**) |
| 仮想 MAC | v1: 0000.0C07.ACxx / v2: 0000.0C9F.Fxxx | 0000.5E00.01xx | 0007.B400.xxyy |
| マルチキャスト | 224.0.0.2 (v1) / **224.0.0.102** (v2) | **224.0.0.18** | 224.0.0.102 |
| 通信 | UDP 1985 | **IP プロトコル 112** | UDP 3222 |
| プリエンプト (既定) | **無効** | **有効** | 無効 |
| 既定プライオリティ | 100 | 100 (アドレス所有者は 255) | 100 |
| Hello / Hold | 3 / 10 秒 | 1 / 約 3 秒 | 3 / 10 秒 |
| IPv6 | HSRP v2 | VRRPv3 | あり |

!!! tip "試験ポイント"
    - HSRP は**既定でプリエンプトが無効**。プライオリティを高くしても `preempt` がなければ奪取しない。
    - HSRP v2 はグループ番号 0〜4095 に対応し、仮想 MAC の形式が v1 と異なる。**v1 と v2 は相互運用不可**。
    - `track` でアップリンク障害時にプライオリティを下げ、`preempt` と組み合わせてフェイルオーバーする。
    - 両ルーターが Active になる場合は、**L2 (VLAN / STP / トランク) の問題**を疑う。

### ハンズオン (HSRP)

R2 を Active (priority 110)、R1 を Standby (priority 100) にします。

=== "R1"

    ```text
    interface GigabitEthernet0/1
     standby version 2
     standby 10 ip 192.168.10.254
     standby 10 priority 100
     standby 10 preempt
     standby 10 authentication md5 key-string CCNP-HSRP
    ```

=== "R2"

    ```text
    track 1 interface GigabitEthernet0/3 line-protocol
    !
    interface GigabitEthernet0/1
     standby version 2
     standby 10 ip 192.168.10.254
     standby 10 priority 110
     standby 10 preempt
     standby 10 authentication md5 key-string CCNP-HSRP
     standby 10 track 1 decrement 20
    ```

!!! note "track の効果"
    R2 の ISP 側 (Gi0/3) がダウンすると、R2 の priority は 110 − 20 = **90** になり、
    R1 (100) が `preempt` により Active になります。

### 確認

```text
R1# show standby brief
R2# show standby brief
R2# show standby GigabitEthernet0/1
PC1# ping 192.168.10.254
```

期待結果: R2 が `Active`、R1 が `Standby`、仮想 IP が `192.168.10.254`、仮想 MAC が `0000.0C9F.F00A` (v2、グループ 10 = 0x00A)。

### 壊して直す (フェイルオーバー)

PC1 から連続 ping を流しつつ、R2 の LAN 側を落とします。

```text
PC1# ping 192.168.10.254 repeat 1000
R2(config)# interface GigabitEthernet0/1
R2(config-if)# shutdown
R1# show standby brief       ← R1 が Active に遷移
R2(config-if)# no shutdown
R2# show standby brief       ← preempt により R2 が Active に復帰
```

- 数パケットの欠落後に VIP への ping が復旧すれば成功です。

**よくある不具合**

| 症状 | 確認 |
|------|------|
| 両方 Active | L2 疎通 (VLAN、STP、トランク)、ACL で 224.0.0.102 を遮断していないか |
| グループ・バージョン不一致 | `show standby` で両端の version / group |
| 認証エラー | `%HSRP-4-BADAUTH` のログ |
| 高プライオリティなのに Standby | `preempt` の有無 |

### 参考: VRRPv3 の設定

HSRP と同一インターフェイスでは併用できないため、ラボには適用しません。

```text
fhrp version vrrp v3
!
interface GigabitEthernet0/1
 vrrp 10 address-family ipv4
  address 192.168.10.254 primary
  priority 110
  preempt
 exit-vrrp
```

```text
show vrrp brief
show vrrp
```

---

## 3.3.d RPF チェック、PIM-SM、IGMP v2/v3、SSM、bidir、MSDP などのマルチキャストプロトコルの説明 { #33d }

!!! abstract "所要 2 分 / 目標"
    マルチキャストの転送モデルと各プロトコルの役割を整理し、
    RPF チェックと基本的な確認コマンドを説明できる。

### 全体像

```mermaid
graph LR
    S["送信元 (S)"] -->|"マルチキャスト<br/>(S,G)"| RP["RP / PIM ルーター"]
    RP -->|"共有ツリー (*,G)"| R["最終ホップルーター"]
    R -->|"IGMP"| H["受信者 (ホスト)"]
    S -.->|"SPT (最短経路)"| R
```

| プロトコル | 役割 |
|------------|------|
| **IGMP** | ホスト ↔ ルーター間。グループ参加・離脱の管理 |
| **PIM** | ルーター間。マルチキャスト配信ツリーの構築 |
| **RPF チェック** | ループ防止。受信 IF が**送信元への最良ユニキャスト経路の出力 IF**と一致するかを検証 |
| **MSDP** | 異なる PIM-SM ドメインの **RP 間で送信元情報 (SA)** を共有 |

### RPF チェック

- パケットの**送信元アドレス**を**ユニキャストルーティングテーブル**で検索する。
- 到達に使う**出力インターフェイス = パケットの受信インターフェイス**なら転送、そうでなければ**破棄**する。
- 共有ツリー (*,G) では送信元ではなく **RP のアドレス**に対して RPF チェックを行う。

```text
show ip rpf 203.0.113.1
show ip rpf <送信元 IP>
mtrace <送信元> <受信者> <グループ>
```

### IGMP v2 / v3

| 項目 | IGMPv2 | IGMPv3 |
|------|--------|--------|
| 送信元の指定 | **不可** (ASM のみ) | **可** (INCLUDE / EXCLUDE → **SSM** に対応) |
| レポート宛先 | 参加するグループ宛 | **224.0.0.22** |
| Leave | あり (Leave Group → 224.0.0.2) | レポートの状態変更で表現 |
| 一般クエリ | 224.0.0.1 宛 | 224.0.0.1 宛 |

```text
interface GigabitEthernet0/1
 ip igmp version 3
```

### PIM-SM、SSM、bidir、MSDP

| 項目 | 説明 |
|------|------|
| **PIM-SM** (Sparse Mode) | 受信者の**明示的な Join** で配信ツリーを構築。**RP** を中心とした共有ツリー (*,G) から、閾値で **最短経路ツリー (S,G) へ切替** |
| RP の決定方法 | **静的** (`ip pim rp-address`) / **Auto-RP** (Cisco) / **BSR** (標準) |
| **SSM** | 受信者が (S,G) を指定。**RP 不要**。既定範囲 **232.0.0.0/8**。IGMPv3 が必須 |
| **BiDir PIM** | 双方向共有ツリー (*,G) のみ。**(S,G) 状態なし**でスケールする。**DF (Designated Forwarder)** を選出。多対多に有効 |
| **MSDP** | **TCP 639**。**Source Active (SA)** メッセージで RP 間の送信元情報を共有。**Anycast RP** 構成でも使用 |

!!! tip "試験ポイント"
    - PIM-SM: 受信者→ RP へ **(*,G) Join**、送信元 → RP へ **Register**、その後 **(S,G) Join** で SPT に切替。
    - SSM は **IGMPv3 + `ip pim ssm default`**。RP は不要。
    - BiDir は **(S,G) を作らない**。`ip pim bidir-enable` が必要。
    - RPF 失敗 = パケット破棄。**ユニキャスト経路が変わるとマルチキャストも変わる**。

### 参考コンフィグ (ラボには適用しません)

```text
! --- 共通 (PIM-SM) ---
ip multicast-routing
interface GigabitEthernet0/0
 ip pim sparse-mode
interface GigabitEthernet0/1
 ip pim sparse-mode
 ip igmp version 3
!
! --- RP (静的) ---
ip pim rp-address 2.2.2.2
!
! --- SSM ---
ip pim ssm default
!
! --- BiDir ---
ip pim bidir-enable
access-list 20 permit 239.100.0.0 0.0.255.255
ip pim rp-address 2.2.2.2 20 bidir
!
! --- MSDP (RP 間) ---
ip msdp peer 192.0.2.2 connect-source Loopback0
```

### 確認コマンド

| コマンド | 内容 |
|----------|------|
| `show ip pim neighbor` | PIM ネイバー |
| `show ip pim rp mapping` | RP とグループの対応 |
| `show ip igmp groups` / `show ip igmp interface` | IGMP の参加状態 |
| `show ip mroute` / `show ip mroute count` | マルチキャストルーティングテーブル (`(*,G)` / `(S,G)`、フラグ、RPF 情報) |
| `show ip rpf <IP>` | RPF インターフェイスとネイバー |
| `show ip msdp peer` / `show ip msdp sa-cache` | MSDP のピアと SA キャッシュ |

!!! example "時間が余ったら (任意)"
    R1 / R2 の Gi0/0・Gi0/1・Lo0 に `ip pim sparse-mode` を設定し、`ip pim rp-address 2.2.2.2` を両方に設定。
    R1 の Gi0/1 に `ip igmp join-group 239.1.1.1` を入れて、`show ip igmp groups` と
    `show ip mroute 239.1.1.1` で `(*,G)` が作られることを確認します
    (事前に両ルーターで `ip multicast-routing` が必要)。
