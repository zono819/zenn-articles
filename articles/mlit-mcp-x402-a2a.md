---
title: "国交省データがエージェントに売れる時代 ── MLIT MCP × x402 × A2A AgentCard で作るエージェント経済の最小構成"
emoji: "🗾"
type: "tech"
topics: ["mlit", "x402", "a2a", "agentcard", "mcp"]
published: false
---

## はじめに

2026年初頭、エージェント経済を動かす「3種の神器」が短期間に出揃った。

| ピース | 何をするか | リリース時期 |
|---|---|---|
| **A2A v1.0 AgentCard** | エージェントがサービスを自律発見する仕組み | 2026年初（Linux Foundation 移管後） |
| **x402 プロトコル** | エージェントが自律決済する仕組み（HTTP 402 + USDC） | 2025年5月（Coinbase）、2026年2月 Stripe 統合 |
| **MLIT 地理空間 MCP サーバー** | 国交省公式の不動産データ入口（無料・公式） | 2026年2月26日 |

この記事では「**不動産ハザードスコア API**」を例題に、この3つを繋ぐ最小構成を解説する。

最終的に実現したいのは次のフローだ。

```
[別エージェント] → A2A で AgentCard を発見
               → skill "hazard-score" を確認
               → x402 で USDC 0.001 を自動決済
               → MLIT MCP サーバー経由でデータ取得・スコアリング
               → ハザードスコアを受け取る
```

「政府公式データ × エージェント自律決済 × エージェント間サービス発見」の組み合わせは、現時点（2026年5月）で日本語記事が存在しない空白地帯だ。

:::message
**免責事項**: 本記事はデモ・技術検証レベルの内容です。MLIT 不動産情報ライブラリの利用規約（加工・再配布条項）については商用利用前に必ず確認してください。本記事のコードを商業サービスに利用する場合、別途国交省への確認が必要です。
:::

---

## 登場人物の整理

### MLIT 不動産情報ライブラリとは

国土交通省が提供するオープンデータポータル。2024年4月から API を一般公開している。

- **データ種類**: 35種類（MCP 経由で30種類アクセス可能）
- **主なデータ**: 取引価格・成約価格、地価公示・地価調査、洪水浸水想定区域、津波浸水想定、土砂災害警戒区域、都市計画情報、周辺施設情報、人口情報
- **料金**: 無料（APIキー申請後5営業日で発行）
- **API キー申請**: https://www.reinfolib.mlit.go.jp/api/request/

### MLIT 地理空間 MCP サーバーとは

2026年2月26日に国交省が GitHub 公開した公式 MCP サーバー。

- リポジトリ: `chirikuuka/mlit-geospatial-mcp`（公式）
- Claude Desktop / Claude Code から自然言語でデータ取得可能
- 現状 α 版（動作保証なし）
- 座標変換（緯度経度 → XYZ タイル）、周辺4タイル並列取得などを内部処理

既存の「試してみた」記事は Zenn・Qiita だけで10件以上ある。**本記事はそこから一歩進んで、このデータを他のエージェントに売る仕組みを作る。**

### A2A プロトコルとは

Google が2025年4月に発表したエージェント間通信の標準プロトコル。2025年6月に Linux Foundation へ寄贈。2026年初頭に v1.0 がリリースされ、**Signed AgentCard**（暗号署名付き）が導入された。

**AgentCard** は `/.well-known/agent-card.json` に置く JSON ファイルで、エージェントの「デジタル名刺」として機能する。他のエージェントがこの URL を取得することで、サービスの存在・機能・エンドポイントを自律的に発見できる。

```json
{
  "name": "MLIT Hazard Score API",
  "description": "国交省データを統合して不動産ハザードスコアを返すAPIエージェント",
  "version": "0.1.0",
  "url": "https://example.com/a2a",
  "skills": [
    {
      "id": "hazard-score",
      "name": "Hazard Score",
      "description": "住所または座標を受け取り、洪水・津波・土砂災害リスクを統合したスコア（0〜10）を返す",
      "tags": ["real-estate", "hazard", "japan", "mlit"],
      "examples": ["東京都江東区亀戸3丁目のハザードスコアを教えて"]
    }
  ],
  "capabilities": {
    "streaming": false
  }
}
```

### x402 プロトコルとは

Coinbase と Cloudflare が主導する、HTTP ネイティブなマイクロペイメント規格。HTTP 402 ステータスコード（"Payment Required"）を使い、USDC などのステーブルコインで即時決済できる。

```
クライアント → POST /hazard-score
サーバー    ← 402 Payment Required（支払い条件を返す）
クライアント → POST /hazard-score + X-PAYMENT: <署名済み決済情報>
サーバー    ← 200 OK + ハザードスコア（チェーン上で決済を検証後）
```

2026年2月に Stripe が x402 をサポートし、クレジットカード払いへの拡張も視野に入った。

### AP2 との違い

AP2（Agent Payments Protocol）はGoogle主導でFIDO標準化中のより広い枠組みで、クレジットカード等の従来決済にも対応する。x402 は AP2 の暗号資産拡張として内包される補完関係。マイクロペイメントの実装としては x402 の方がシンプルで、本記事でも x402 を使う。

---

## 全体アーキテクチャ

```
┌─────────────────────────────────────────────────────┐
│  クライアントエージェント（例: 不動産提案 AI）           │
└──────────────────┬──────────────────────────────────┘
                   │ 1. GET /.well-known/agent-card.json
                   │ 2. skill "hazard-score" を確認
                   ▼
┌─────────────────────────────────────────────────────┐
│  ハザードスコア API サーバー（本記事で作るもの）         │
│                                                     │
│  ├── A2A エンドポイント (/a2a)                       │
│  ├── AgentCard 公開 (/.well-known/agent-card.json)  │
│  ├── x402 ミドルウェア（決済検証）                    │
│  └── MLIT MCP クライアント                          │
└──────────────────┬──────────────────────────────────┘
                   │ 3. MLIT API 呼び出し
                   ▼
┌─────────────────────────────────────────────────────┐
│  MLIT 不動産情報ライブラリ API（無料・国交省公式）       │
│  ├── 洪水浸水想定区域                                │
│  ├── 津波浸水想定                                    │
│  ├── 土砂災害警戒区域                                │
│  └── 地価公示データ                                  │
└─────────────────────────────────────────────────────┘
```

**決済フロー（x402）:**

```
クライアント                 APIサーバー
    │                           │
    │── POST /a2a ─────────────>│
    │                           │（未払いを検知）
    │<─ 402 + 支払い条件 ────────│
    │                           │
    │（USDC 0.001 を署名）        │
    │── POST /a2a + X-PAYMENT ──>│
    │                           │（ブロックチェーンで検証）
    │                           │（MLIT API を呼び出し）
    │<─ 200 + ハザードスコア ─────│
```

---

## Step 1: MLIT MCP サーバーを繋ぐ

まず、MLIT 地理空間 MCP サーバーを使ってデータが取れることを確認する。

```bash
# MLIT 地理空間 MCP サーバーをクローン
git clone https://github.com/chirikuuka/mlit-geospatial-mcp
cd mlit-geospatial-mcp

# 依存インストール
pip install -r requirements.txt

# .env に API キーを設定
echo "REINFOLIB_API_KEY=your_api_key_here" > .env
```

Claude Desktop の `claude_desktop_config.json` に追加:

```json
{
  "mcpServers": {
    "mlit-geospatial": {
      "command": "python",
      "args": ["/path/to/mlit-geospatial-mcp/server.py"],
      "env": {
        "REINFOLIB_API_KEY": "your_api_key_here"
      }
    }
  }
}
```

Claude Desktop を再起動すると、自然言語でデータを取得できるようになる。

**動作確認の例:**
```
「東京都江東区亀戸3丁目の洪水リスクを教えて」
→ 洪水浸水想定深さ: 0.5〜3m
  津波浸水リスク: 低（標高3m以上）
  土砂災害警戒区域: 対象外
```

この段階では**完全無料**。MLIT API が無料なので、MCP 経由のデータ取得にコストはかからない。

---

## Step 2: x402 エンドポイントを被せる

「無料で取れるデータを、なぜ x402 で有料にするか？」

**答えは「加工への課金」だ。**

- 洪水・津波・土砂災害の3データを統合して**単一スコア（0〜10）**に変換する
- スコアの算出ロジックに技術的価値がある
- 不動産融資・保険引受・投資判断に直接使えるフォーマットで返す

生の MLIT データを取ってスコアに変換する手間・ロジックに対して課金する。

```python
# TBD: FastAPI + x402 ミドルウェアの実装例
# 実装が固まり次第追記予定

# イメージ:
# from fastapi import FastAPI
# from x402 import X402Middleware
#
# app = FastAPI()
# app.add_middleware(X402Middleware, price_usdc=0.001, wallet="0x...")
#
# @app.post("/hazard-score")
# async def hazard_score(address: str):
#     # 1. 住所 → 座標変換
#     # 2. MLIT API から洪水・津波・土砂データ取得
#     # 3. スコア算出（加重平均等）
#     # 4. スコアを返す
#     pass
```

:::message alert
**TBD**: x402 Python ライブラリの具体的な使い方は実装検証後に追記。現在 `coinbase/x402` の Python SDK を調査中。
:::

---

## Step 3: AgentCard を公開する

`/.well-known/agent-card.json` に AgentCard を配置する。他のエージェントはこの URL を取得することでサービスを自律発見できる。

```json
{
  "name": "MLIT Hazard Score API",
  "description": "国土交通省の公式データを統合した不動産ハザードスコアAPI。住所を入力すると洪水・津波・土砂災害リスクを0〜10のスコアで返す。x402でUSDCによるマイクロペイメントに対応。",
  "version": "0.1.0",
  "url": "https://example.com/a2a",
  "provider": {
    "organization": "TBD"
  },
  "skills": [
    {
      "id": "hazard-score",
      "name": "不動産ハザードスコア",
      "description": "日本の住所または緯度経度を受け取り、MLIT公式データに基づくハザードスコア（0〜10）を返す。スコアが高いほどリスクが高い。",
      "tags": ["real-estate", "hazard", "japan", "mlit", "flood", "tsunami", "landslide"],
      "examples": [
        "東京都江東区亀戸3丁目のハザードスコアを教えて",
        "35.6894,139.6917 のリスク評価をして"
      ],
      "input_modes": ["text/plain", "application/json"],
      "output_modes": ["application/json"]
    }
  ],
  "capabilities": {
    "streaming": false,
    "x402": {
      "supported": true,
      "price_usdc": "0.001",
      "network": "base"
    }
  }
}
```

**A2A v1.0 では AgentCard に暗号署名を付与できる**（Signed AgentCard）。これにより、クライアントエージェントは「この AgentCard がドメイン所有者によって正式に発行されたもの」を検証できる。

```python
# TBD: Signed AgentCard の生成コード
# A2A Python SDK を使った署名付き AgentCard 生成は実装後に追記
```

---

## 動作デモ

:::message
**TBD**: 実装完了後にスクリーンショットまたはアスキーアートのデモを追加予定。
:::

想定される動作シナリオ:

```
# 別のエージェント（例: 不動産提案 AI）が自律的に動く

エージェント: 「江東区の物件を提案する前にリスクチェックしよう」
→ A2A で hazard-score スキルを持つエージェントを検索
→ https://example.com/.well-known/agent-card.json を取得
→ skill "hazard-score" を確認、価格 0.001 USDC を確認
→ 自律的に USDC 0.001 を支払い
→ 「東京都江東区亀戸3丁目」のスコアを取得
→ スコア: 7.2/10（洪水リスク高）
→ ユーザーへの提案に「洪水リスク高エリア」の注記を追加
```

このフローで**人間の介入なし**に、エージェントがデータを発見・決済・利用する。

---

## 新規性と既存との比較

| 観点 | 既存 | 本記事 |
|---|---|---|
| MLIT データ利用 | 「試してみた」記事が10件以上 | x402 有料エンドポイントとして公開 |
| データ発見性 | 人間が URL を直打ち | A2A AgentCard で他エージェントが自律発見 |
| 決済方式 | 無料（API キー） | x402 マイクロペイメント（USDC） |
| 組み合わせ | 各要素は個別に存在 | **3つを繋いだ実装は国内初（2026年5月時点）** |

**日本語記事で参照すべき先行事例:**

- x402 日本語解説: [x402: インターネットネイティブな決済の新標準](https://zenn.dev/aki_on/articles/498effc7c4f18b)
- MLIT MCP 試してみた: [国交省の不動産情報ライブラリMCPサーバを試す](https://zenn.dev/rescuenow/articles/fe5fc6f226cea7)
- A2A 公式仕様: https://a2a-protocol.org/latest/

**英語圏の参照先:**

- A2A × x402 公式実装: https://github.com/google-agentic-commerce/a2a-x402
- ArXiv 論文（A2A + x402）: https://arxiv.org/html/2507.19550v1

---

## 今後の展望

### AP2 への移行パス

現在の x402 は USDC on Base が主な決済手段だが、AP2（Agent Payments Protocol）が普及すれば Stripe 経由でクレジットカード払いにも対応できる。AP2 は x402 を暗号資産拡張として内包しているため、実装の移行コストは小さい。

### 具体的なユースケース（TBD）

実装が進んだ段階で以下を追記予定:
- 分譲マンション価格妥当性チェック（地価 × ハザード × 周辺施設の統合評価）
- 不動産融資審査 AI エージェントへの組み込み
- 保険引受自動化への応用

### MLIT データ利用規約について

本記事のアーキテクチャを商用サービスに応用する場合、MLIT 不動産情報ライブラリの利用規約における「加工・再配布」条項を事前に確認すること。現状 α 版サービスのため、正式な商用利用については国交省への問い合わせを推奨する。

---

## まとめ

- **MLIT 地理空間 MCP サーバー**（無料・公式）を x402 有料エンドポイントに変換する「加工への課金」モデルを提案した
- **A2A AgentCard** を使うことで、他のエージェントがサービスを自律発見・利用できる
- **x402** により、エージェントが人間の介入なしに自律決済できる
- この3つの組み合わせは 2026年5月時点で日本語・英語ともに実装記事が存在しない空白地帯

コードは GitHub に公開予定（TBD）。フィードバックや共同実装の提案はコメントまたは X (Twitter) まで。

---

*本記事は調査フェーズの内容をまとめたものです。実装フェーズの進捗に合わせて随時更新します。*
