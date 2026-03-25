---
title: "ChatGPTやClaudeから不動産データを取得する方法（FUDOSAN DB × MCP）"
emoji: "🏠"
type: "tech"
topics: ["mcp", "claude", "chatgpt", "不動産", "api"]
published: false
---

## やりたいこと

ChatGPTやClaudeに「渋谷区の1LDK、築5年の家賃相場は？」と聞いたら、実データに基づいた回答が返ってくる。

これ、もうできます。

FUDOSAN DBを接続するだけで、AIが不動産データにアクセスできるようになります。取引価格730万件、地価公示、賃料推定AIまで。

接続方法はAIクライアントごとに異なるので、それぞれ解説します。

## FUDOSAN DBとは

国交省の不動産情報ライブラリ（reinfolib）のデータを構造化して提供するAPIサービスです。

reinfolib APIを直接使うと、PBFデコード、座標変換、データ正規化といった前処理に数週間かかります。FUDOSAN DBはこれらの前処理済みデータをJSON API / MCPで提供しています。

- 取引価格データ 730万件
- 地価公示・地価調査
- 賃料推定AI（誤差2%台）
- エリア分析・ランキング・スクリーナー
- 申請不要、APIキー即時発行、無料プランあり

https://fudosandb.jp

## 接続方法

### 1. Claude.ai（Web版）

Claude.aiのMCPインテグレーション機能を使います。

1. [Claude.ai](https://claude.ai) にログイン
2. 設定 → Integrations → 「Add more integrations」
3. FUDOSAN DBを検索して「Connect」
4. OAuthで認証（APIキーが自動設定されます）

接続後は、通常のチャットで不動産データについて質問するだけです。

### 2. Claude Desktop

claude_desktop_config.json に以下を追加して再起動します。

```json
{
  "mcpServers": {
    "fudosandb": {
      "command": "npx",
      "args": [
        "@anthropic-ai/mcp-remote",
        "https://fudosandb.jp/mcp",
        "--header",
        "X-API-Key: YOUR_API_KEY"
      ]
    }
  }
}
```

Node.js 18以上が必要です。APIキーは https://fudosandb.jp/developers で即時発行できます。

### 3. Claude Code

settings.jsonまたはプロジェクトの `.mcp.json` に追加します。

```json
{
  "mcpServers": {
    "fudosandb": {
      "command": "npx",
      "args": [
        "@anthropic-ai/mcp-remote",
        "https://fudosandb.jp/mcp",
        "--header",
        "X-API-Key: YOUR_API_KEY"
      ]
    }
  }
}
```

追加後、Claude Codeを再起動すると「渋谷区の家賃相場を調べて」のような指示でデータを取得できます。

### 4. ChatGPT（Custom GPTs）

ChatGPTはMCPに直接対応していないため、REST APIをActionsとして設定します。

1. [ChatGPT](https://chat.openai.com) → GPTsを探す → 「GPTを作成」
2. 「Configure」タブ → 下部の「Actions」→「Create new action」
3. 以下のOpenAPIスキーマを貼り付け:

```yaml
openapi: "3.0.0"
info:
  title: FUDOSAN DB API
  version: "1.0"
servers:
  - url: https://fudosandb.jp
paths:
  /v1/area-profile/{municipality_code}:
    get:
      operationId: getAreaProfile
      summary: エリアの総合プロファイルを取得
      parameters:
        - name: municipality_code
          in: path
          required: true
          schema:
            type: string
          description: "市区町村コード（5桁。例: 13113=渋谷区）"
      responses:
        "200":
          description: エリアプロファイル
  /v1/estimate-rent:
    post:
      operationId: estimateRent
      summary: AI賃料推定
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [municipality_code, area_m2]
              properties:
                municipality_code:
                  type: string
                  description: "市区町村コード（例: 13113）"
                area_m2:
                  type: number
                  description: "面積（m²）"
                build_age_years:
                  type: integer
                  description: "築年数"
                layout:
                  type: string
                  description: "間取り（例: 1K, 1LDK, 2DK）"
      responses:
        "200":
          description: 賃料推定結果
  /v1/rankings:
    get:
      operationId: getRankings
      summary: 不動産指標ランキング
      parameters:
        - name: metric
          in: query
          required: true
          schema:
            type: string
            enum: [land_price_high, land_price_growth, transaction_volume, condo_price_high]
          description: "ランキング指標"
      responses:
        "200":
          description: ランキング結果
```

4. 「Authentication」→ 「API Key」→ Header名: `X-API-Key`、値にAPIキーを入力
5. 保存して公開

これでChatGPTから「渋谷区のエリア情報を取得して」と指示できます。

:::message
上記は主要3エンドポイントのみのサンプルです。全エンドポイントのOpenAPIスキーマは [APIドキュメント](https://fudosandb.jp/docs/api) を参照してください。
:::

## 使ってみる

どのクライアントからでも、以下のような質問が使えます。

**エリアの相場を調べる:**
> 「港区と世田谷区の不動産取引価格を比較して」

**賃料を推定する:**
> 「渋谷区で25m²、築10年の1Kの家賃を推定して」

**地価トレンドを見る:**
> 「千代田区の過去5年の地価推移を教えて」

**条件でエリアを絞り込む:**
> 「東京23区で地価上昇率が高い順にランキングして」

**投資シミュレーション:**
> 「福岡市中央区の1LDK、購入価格2000万円で収益シミュレーションして」

## 使えるMCPツール一覧

MCP接続（Claude.ai / Claude Desktop / Claude Code）で使えるツールです。

| ツール | できること |
|-------|----------|
| `search_areas` | エリア検索（市区町村名・都道府県で絞り込み） |
| `get_area_profile` | エリアの総合プロファイル（人口・施設・地価・災害リスク） |
| `get_price_trends` | 不動産価格の四半期推移 |
| `get_land_price_trends` | 地価公示・地価調査の年次推移 |
| `get_rankings` | 指標別ランキング（地価・取引量・マンション価格等） |
| `estimate_rent` | AI賃料推定（住所・面積・築年数から推定） |
| `simulate_yield` | 投資収益シミュレーション |
| `list_municipalities` | 市区町村一覧 |

## REST APIとしても使える

MCPやCustom GPTsを使わず、コードから直接叩くこともできます。

```python
import requests

# エリアプロファイルを取得
resp = requests.get(
    "https://fudosandb.jp/v1/area-profile/13113",
    headers={"X-API-Key": "YOUR_API_KEY"}
)
data = resp.json()
# 渋谷区: 取引中央値、地価、人口、施設数、災害リスク
```

```python
# 賃料推定
resp = requests.post(
    "https://fudosandb.jp/v1/estimate-rent",
    headers={"X-API-Key": "YOUR_API_KEY"},
    json={
        "municipality_code": "13113",
        "area_m2": 25,
        "build_age_years": 10,
        "layout": "1K"
    }
)
# => {"estimated_rent_yen": 112000, "confidence": {"mape_percent": 2.1}}
```

APIキーは https://fudosandb.jp/developers で即時発行できます。無料プランで月100リクエスト。

## reinfolib APIとの違い

| | reinfolib API | FUDOSAN DB |
|---|---|---|
| データ形式 | GeoJSON / PBF | JSON（構造化済み） |
| APIキー発行 | 申請→5営業日 | 即時発行 |
| 前処理 | 自前で必要 | 不要 |
| AI接続 | 国交省α版MCP（25 API） | MCP + REST API + Custom GPTs |
| 賃料推定 | なし | あり（誤差2%台） |
| 料金 | 無料 | 無料プランあり |

reinfolib APIは生データを自分で加工したい場合に向いています。FUDOSAN DBは構造化済みのデータをすぐ使いたい、AIから接続したいという場合に向いています。用途に応じて使い分けてください。

## まとめ

| クライアント | 接続方法 | 難易度 |
|------------|---------|--------|
| Claude.ai | Integrations から追加 | 最も簡単 |
| Claude Desktop | claude_desktop_config.json に3行追加 | 簡単 |
| Claude Code | settings.json に追加 | 簡単 |
| ChatGPT | Custom GPTs の Actions に設定 | やや手間 |
| コードから直接 | REST API を叩く | 自由度が高い |

どの方法でも、AIに不動産データを接続すると「このエリアの相場は？」「投資利回りは？」といった質問に、実データに基づいた回答が返ってきます。

https://fudosandb.jp

---

*FUDOSAN DBは[カボシア株式会社](https://cabocia.jp)が運営しています。データソース: 国土交通省 不動産情報ライブラリ*
