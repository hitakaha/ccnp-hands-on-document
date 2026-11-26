# まとめ・確認問題

!!! abstract "所要 1 分"
    NTP の同期確認と、確認コマンド一覧で全体を振り返ります。

## NTP 同期の確認 (3.3.a の続き)

```text
R2# show ntp status
R3# show ntp associations
```

`Clock is synchronized` と表示されれば完了です。

## 総合確認 (エンドツーエンド)

```text
PC1# ping 192.168.10.254      ← HSRP VIP (3.3.c)
PC1# ping 203.0.113.1         ← PAT + eBGP + OSPF デフォルトルート (3.2 / 3.3.b)
R2# show ip nat translations
R1# show ip route ospf        ← 要約 10.3.0.0/23 とデフォルトルート (3.2.b)
R2# show ip bgp summary       ← eBGP Established (3.2.c)
SW1# show etherchannel summary   ← Po1(SU) (3.1.b)
SW3# show spanning-tree mst   ← MST (3.1.c)
```

## 確認コマンド早見表

| 試験範囲 | 主な確認コマンド |
|----------|------------------|
| 3.1.a トランク | `show interfaces trunk` / `show interfaces <IF> switchport` / `show dtp interface` |
| 3.1.b EtherChannel | `show etherchannel summary` / `show lacp neighbor` / `show pagp neighbor` / `show etherchannel detail` |
| 3.1.c STP | `show spanning-tree [vlan N]` / `show spanning-tree mst [configuration]` / `show spanning-tree summary` / `show spanning-tree inconsistentports` |
| 3.2.a EIGRP / OSPF | `show ip eigrp topology` / `show ip ospf database` / `show ip route` |
| 3.2.b OSPF | `show ip ospf neighbor` / `show ip ospf interface [brief]` / `show ip protocols` / `show ospfv3 neighbor` |
| 3.2.c BGP | `show ip bgp summary` / `show ip bgp [prefix]` / `show ip bgp neighbors` |
| 3.2.d PBR | `show ip policy` / `show route-map` / `debug ip policy` |
| 3.3.a NTP / PTP | `show ntp status` / `show ntp associations` / `show ptp clock` |
| 3.3.b NAT | `show ip nat translations` / `show ip nat statistics` / `clear ip nat translation *` |
| 3.3.c FHRP | `show standby [brief]` / `show vrrp [brief]` |
| 3.3.d マルチキャスト | `show ip mroute` / `show ip pim neighbor` / `show ip igmp groups` / `show ip rpf` |

## 確認問題

回答は折りたたまれています。クリックして確認してください。

**Q1.** 両端が `switchport mode dynamic auto` のポート間は、どうなりますか。

??? success "答え"
    **トランクにならず、アクセスポートのまま**です。`auto` は相手から DTP で要求されるのを待つだけです。
    片側を `desirable` または `trunk` にすればトランクになります。

**Q2.** SW1 が `channel-group 1 mode active`、SW2 が `channel-group 1 mode on` の場合は。

??? success "答え"
    **EtherChannel は形成されません**。LACP と静的 (on) は互換性がありません。
    `active` ↔ `active` / `passive`、または `on` ↔ `on` に合わせます。

**Q3.** Root Guard と Loop Guard の使い分けは。

??? success "答え"
    **Root Guard**: 指定ポートで、**上位 BPDU を受信してルートが変わるのを防ぐ**。ディストリビューション → アクセス方向に設定。
    **Loop Guard**: 非指定ポート (ルート / Alternate) で、**BPDU が途絶えたときの誤 Forwarding を防ぐ**。アップリンク側に設定。

**Q4.** MST でスイッチ間のリージョンが分かれる条件は。

??? success "答え"
    **リージョン名、リビジョン番号、VLAN とインスタンスのマッピング**のいずれかが異なる場合です。

**Q5.** EIGRP の Feasible Successor になる条件は。

??? success "答え"
    隣接ルーターの **Reported Distance (RD) が、自ルーターの Feasible Distance (FD) より小さい**ことです (フィージビリティ条件)。

**Q6.** 同じプレフィックスに OSPF の `O IA` 経路 (コスト 20) と `O E2` 経路 (コスト 10) がある場合、どちらが選ばれますか。

??? success "答え"
    **`O IA`** です。OSPF では経路タイプの優先順 (エリア内 > エリア間 > E1 > E2) がコスト比較より先に評価されます。

**Q7.** OSPF ネイバーが `EXSTART` で止まる典型的な原因は。

??? success "答え"
    **インターフェイスの MTU 不一致**です。`show ip ospf interface` / `show interface` で両端の MTU を確認します。

**Q8.** BGP のベストパス選択で、最初に評価される属性は。

??? success "答え"
    **Weight** (Cisco 独自、ルーターのローカルのみ有効) です。次に Local Preference、ローカル生成、AS_PATH 長、Origin、MED の順です。

**Q9.** PBR はどこに、どの方向で適用しますか。

??? success "答え"
    **入力インターフェイス**に `ip policy route-map <名前>` で適用します。ルーター自身が送信するトラフィックには `ip local policy route-map` を使います。

**Q10.** PAT で複数の内部ホストを 1 つのグローバルアドレスに変換するとき、何で識別しますか。

??? success "答え"
    **ポート番号 (ICMP の場合は ID)** です。`ip nat inside source list <ACL> interface <IF> overload` で設定します。

**Q11.** HSRP で、プライオリティが高いルーターを再起動後に Active に戻すために必要な設定は。

??? success "答え"
    **`standby <group> preempt`** です。HSRP は既定ではプリエンプトが無効です (VRRP は既定で有効)。

**Q12.** SSM の特徴と必要な IGMP バージョンは。

??? success "答え"
    受信者が **(S,G)** で参加するため **RP が不要**。既定範囲は **232.0.0.0/8**。**IGMPv3** が必須です。

## 次のステップ

- 各トピックの「壊して直す」を、**ログの文字列と `show` の出力から原因を即答できる**まで繰り返す。
- 発展: eBGP の 2 パス構成 (Local Preference / AS-path prepend)、OSPF のスタブ/NSSA、EIGRP の `variance`、
  VRRPv3、MST の複数インスタンスの負荷分散。
- 試験では**トラブルシュート**と**出力の読み取り**が頻出です。コマンド出力の読み方に慣れましょう。
