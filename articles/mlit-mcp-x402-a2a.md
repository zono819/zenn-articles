---
title: "国交省データがエージェントに売れる時代──MLIT MCP × x402 × A2A AgentCard で作るエージェント経済の最小構成"
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

MLIT の生データをそのまま返すのではなく、複数 API を統合・スコアリングして**直接判断に使えるフォーマット**に変換する。その加工ロジックと統合コストに対して課金する。

### 公開スキル一覧

| Endpoint | 価格 (USDC) | 使用 MLIT API | 概要 |
|---|---|---|---|
| `POST /skills/condo-price-validity` | $0.02 | 取引価格・地価・都市計画・用途地域・駅乗降客数 | 分譲マンション価格の妥当性スコア（0〜100） |
| `POST /skills/hazard-score` | $0.01 | 液状化・洪水・高潮・津波・土砂災害 | 複合ハザードスコア（0〜100、高いほどリスク大） |
| `POST /skills/urban-context` | $0.005 | 都市計画区域・用途地域・立地適正化・将来人口・防火地域 | 都市計画コンテキスト一式 |
| `POST /skills/land-price-trend` | $0.01 | 地価公示・地価調査 | 地価トレンド（CAGR・年別中央値） |

### ユースケース例：分譲マンション価格妥当性チェック（`condo-price-validity`）

「この物件、8500万円は適正か？」をエージェントが自律的に判断したいとき。

**リクエスト:**

```json
POST /skills/condo-price-validity
Content-Type: application/json
X-PAYMENT: <x402署名済み決済情報 / 0.02 USDC>

{
  "lat": 35.6581,
  "lon": 139.7017,
  "asking_price_jpy": 85000000,
  "area_sqm": 65.0
}
```

**レスポンス:**

```json
{
  "skill": "condo-price-validity",
  "score": 78,
  "breakdown": {
    "price_vs_transactions": {
      "score": 38, "max": 50,
      "detail": "Asking ¥1,307,692/sqm is within ±35% of median ¥1,100,000/sqm (12 transactions)"
    },
    "land_price_alignment": {
      "score": 20, "max": 20,
      "detail": "Price/sqm (¥1,307,692) vs land price (¥650,000) ratio 2.0x — reasonable"
    },
    "zoning_quality": {
      "score": 15, "max": 15,
      "detail": "Favorable zoning: 第一種住居地域"
    },
    "transit_access": {
      "score": 12, "max": 15,
      "detail": "3 station(s) nearby, highest ridership 45,200/day — good transit"
    }
  },
  "data_coverage": {
    "transaction_count": 12,
    "land_price_points": 4,
    "zoning_found": true,
    "station_count": 3
  }
}
```

**スコア解釈:**

| スコア | 意味 |
|---|---|
| 80〜100 | 市場データに十分裏付けられた価格 |
| 60〜79 | おおむね妥当、軽微な懸念あり |
| 40〜59 | 一部ファクターに注意が必要 |
| 0〜39 | 価格が市場と大きくかい離している可能性 |

スコア78 → 「おおむね妥当」。取引価格比較・地価整合性・用途地域・駅アクセスの4ファクターが内訳として返るため、**エージェントが根拠付きで判断を下せる**。

同様に `hazard-score` では液状化・洪水・高潮・津波・土砂災害の5ハザードを統合し、`land-price-trend` では指定年数の地価推移（CAGR付き）を返す。

---

## Step 3: AgentCard を公開する

`/.well-known/agent-card.json` に AgentCard を配置する。他のエージェントはこの URL を取得することでサービスを自律発見できる。

### `createX402AgentCard()` ヘルパーを使った生成

`a2a-x402` ライブラリ（`google-agentic-commerce/a2a-x402`）が提供する `createX402AgentCard()` ヘルパーを使うと、スキルごとの x402 決済情報を AgentCard に埋め込める。

```typescript
import { createX402AgentCard } from "@google-agentic-commerce/a2a-x402";

const card = createX402AgentCard({
  name: "MLIT Real Estate Intelligence API",
  description:
    "国土交通省の公式地理空間データを統合した不動産分析 API。" +
    "価格妥当性・ハザードリスク・都市計画コンテキスト・地価トレンドを x402 マイクロペイメントで提供。",
  version: "0.1.0",
  url: "https://example.com/a2a",
  skills: [
    {
      id: "condo-price-validity",
      name: "分譲マンション価格妥当性チェック",
      description:
        "緯度経度・物件価格・専有面積を受け取り、MLIT取引価格・地価・用途地域・駅アクセスの4ファクターで妥当性スコア(0-100)を返す。",
      tags: ["real-estate", "price", "japan", "mlit", "condo"],
      examples: ["lat:35.6581, lon:139.7017, 8500万円, 65㎡ の価格は妥当か"],
      x402: { priceUsdc: "0.02", network: "base" },
    },
    {
      id: "hazard-score",
      name: "不動産ハザードスコア",
      description:
        "座標を受け取り、液状化・洪水・高潮・津波・土砂災害の5ハザードを統合したリスクスコア(0-100)を返す。スコアが高いほどリスク大。",
      tags: ["real-estate", "hazard", "japan", "mlit", "flood", "tsunami"],
      examples: ["lat:35.6581, lon:139.7017 のハザードスコアを教えて"],
      x402: { priceUsdc: "0.01", network: "base" },
    },
    {
      id: "urban-context",
      name: "都市計画コンテキスト",
      description:
        "座標を受け取り、都市計画区域・用途地域・立地適正化計画・防火地域・将来人口トレンドを一括返却する。",
      tags: ["real-estate", "urban-plan", "japan", "mlit", "zoning"],
      examples: ["lat:35.6581, lon:139.7017 の用途地域と人口トレンドを教えて"],
      x402: { priceUsdc: "0.005", network: "base" },
    },
    {
      id: "land-price-trend",
      name: "地価トレンド",
      description:
        "座標と年リストを受け取り、地価公示・地価調査データから年別中央値と CAGR を返す。",
      tags: ["real-estate", "land-price", "japan", "mlit", "trend"],
      examples: ["lat:35.6581, lon:139.7017 の2021〜2025年の地価トレンドは？"],
      x402: { priceUsdc: "0.01", network: "base" },
    },
  ],
});
```

### 生成される AgentCard（抜粋）

```json
{
  "name": "MLIT Real Estate Intelligence API",
  "description": "国土交通省の公式地理空間データを統合した不動産分析 API。...",
  "version": "0.1.0",
  "url": "https://example.com/a2a",
  "skills": [
    {
      "id": "condo-price-validity",
      "name": "分譲マンション価格妥当性チェック",
      "tags": ["real-estate", "price", "japan", "mlit", "condo"],
      "x402": { "priceUsdc": "0.02", "network": "base" }
    },
    {
      "id": "hazard-score",
      "name": "不動産ハザードスコア",
      "tags": ["real-estate", "hazard", "japan", "mlit", "flood", "tsunami"],
      "x402": { "priceUsdc": "0.01", "network": "base" }
    },
    {
      "id": "urban-context",
      "name": "都市計画コンテキスト",
      "tags": ["real-estate", "urban-plan", "japan", "mlit", "zoning"],
      "x402": { "priceUsdc": "0.005", "network": "base" }
    },
    {
      "id": "land-price-trend",
      "name": "地価トレンド",
      "tags": ["real-estate", "land-price", "japan", "mlit", "trend"],
      "x402": { "priceUsdc": "0.01", "network": "base" }
    }
  ],
  "capabilities": { "streaming": false }
}
```

**A2A v1.0 では AgentCard に暗号署名を付与できる**（Signed AgentCard）。これにより、クライアントエージェントは「この AgentCard がドメイン所有者によって正式に発行されたもの」を検証できる。`createX402AgentCard()` は署名用のフックも提供しており、秘密鍵を渡すだけで Signed AgentCard を生成できる。

---

## 動作デモ

実際の動作シナリオ（不動産提案 AI がユーザーに物件を提案する場面）:

```
# 不動産提案エージェントが自律的に動く

1. [発見] GET https://example.com/.well-known/agent-card.json
   → 4スキルを発見。"condo-price-validity" ($0.02) と "hazard-score" ($0.01) を使うと判断

2. [価格妥当性チェック] POST /skills/condo-price-validity
   → 402 Payment Required（$0.02 USDC）
   → X-PAYMENT ヘッダーで自律的に署名・送信
   → score: 78 / "おおむね妥当、取引価格の中央値±35%以内"

3. [ハザードチェック] POST /skills/hazard-score
   → 402 Payment Required（$0.01 USDC）
   → X-PAYMENT ヘッダーで自律的に支払い
   → score: 22 / 内訳: 液状化medium、洪水medium、津波none、土砂none

4. [提案生成] ユーザーへの出力:
   「江東区 35.6581,139.7017 の物件（8,500万円・65㎡）の分析結果:
    - 価格妥当性スコア: 78/100（おおむね妥当）
      ・周辺12件の取引価格中央値 ¥1,100,000/㎡ に対し ¥1,307,692/㎡（+18.9%）
      ・地価との整合性: 良好（地価比2.0x）
    - ハザードスコア: 22/100（中程度リスク）
      ・液状化リスク: 中（可能性がある）
      ・洪水浸水深: 0.5〜3m（ランク2）
      ・津波・高潮・土砂: リスクなし
    ⚠️ 液状化・洪水リスクを踏まえた保険・修繕積立の確認を推奨します」

合計コスト: $0.03 USDC（人間の介入ゼロ）
```

このフローで**人間の介入なし**に、エージェントがデータを発見・決済・利用して**根拠付きの提案**を生成する。

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

### 具体的なユースケース

現在実装済みの4スキルで対応できるユースケース:

- **分譲マンション価格妥当性チェック** → `condo-price-validity`（$0.02）
  取引価格・地価・用途地域・駅アクセスの4ファクターで100点スコアリング
- **不動産融資審査 AI エージェントへの組み込み** → `hazard-score` + `urban-context`（$0.015）
  担保不動産のリスクサマリーを自動生成、審査フローに組み込み可能
- **保険引受自動化** → `hazard-score`（$0.01）
  5ハザードのスコアを保険料算定ロジックへ自動フィード
- **投資判断支援** → `land-price-trend` + `urban-context`（$0.015）
  地価CAGR × 人口動態で中長期の資産価値を評価

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
