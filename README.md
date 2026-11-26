# CCNP ENCOR ハンズオン (30分) - MkDocs

CCNP ENCOR 350-401「3.0 インフラストラクチャ」(3.1 レイヤ 2 / 3.2 レイヤ 3 / 3.3 IP サービス) を
30 分でカバーするハンズオンコースの MkDocs (Material テーマ) プロジェクトです。

## 使い方

```bash
# 初回のみ
python3 -m venv venv
venv/bin/pip install -r requirements.txt

# ローカルプレビュー (http://127.0.0.1:8000)
venv/bin/mkdocs serve

# 静的サイトの生成 (site/)
venv/bin/mkdocs build --strict
```

## 構成

| ファイル | 内容 |
|----------|------|
| `mkdocs.yml` | MkDocs 設定 |
| `docs/index.md` | コース概要・タイムテーブル |
| `docs/00-lab-setup.md` | トポロジー・アドレス設計・初期コンフィグ |
| `docs/31-layer2.md` | 3.1.a〜3.1.c (トランク / EtherChannel / STP) |
| `docs/32-layer3.md` | 3.2.a〜3.2.d (EIGRP vs OSPF / OSPFv2・v3 / eBGP / PBR) |
| `docs/33-ip-services.md` | 3.3.a〜3.3.d (NTP・PTP / NAT・PAT / HSRP・VRRP / マルチキャスト) |
| `docs/99-summary.md` | まとめ・確認コマンド・確認問題 |
