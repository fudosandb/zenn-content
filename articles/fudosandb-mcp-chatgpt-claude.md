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

FUDOSAN DBをMCPで接続するだけで、AIが不動産データにアクセスできるようになります。取引価格730万件、地価公示、賃料推定AIまで。

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

FUDOSAN DBはMCPのOAuth 2.1認証に対応しています。ChatGPT、Claude.ai、Claude Desktop、Claude Code、どれからでもMCPで接続できます。

### Claude.ai（Web版）

1. [Claude.ai](https://claude.ai) にログイン
2. チャット画面の下部にある検索アイコン（MCP Integrations）をクリック
3. 「FUDOSAN DB」を検索して「Add」
4. Googleアカウントで認証（OAuthフロー）

認証が完了すると、通常のチャットで不動産データについて質問できるようになります。

### ChatGPT

ChatGPTもMCPコネクタに対応しています。

1. [ChatGPT](https://chatgpt.com) にログイン
2. チャット画面から MCP Server を追加
3. URL に `https://fudosandb.jp/mcp` を入力
4. Googleアカウントで認証（OAuthフロー）

Claude.aiと同じOAuth認証で、セットアップの手順はほぼ同じです。

### Claude Desktop

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

### Claude Code

`~/.claude/settings.json` に追加します。

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

## 使ってみる

どのクライアントからでも、自然言語で質問するだけです。

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

MCPツールが用意されているので、AIが適切なツールを選んでデータを取得・整理してくれます。

## 使えるMCPツール一覧

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

MCP以外に、コードから直接REST APIを叩くこともできます。

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
| AI接続 | 国交省α版MCP（25 API） | MCP対応（ChatGPT / Claude / Claude Code） |
| 賃料推定 | なし | あり（誤差2%台） |
| 料金 | 無料 | 無料プランあり |

reinfolib APIは生データを自分で加工したい場合に向いています。FUDOSAN DBは構造化済みのデータをすぐ使いたい、AIから接続したいという場合に向いています。

## まとめ

| クライアント | 接続方法 | 認証 |
|------------|---------|------|
| Claude.ai | MCP Integrations から追加 | OAuth（Google） |
| ChatGPT | MCP Server URLを追加 | OAuth（Google） |
| Claude Desktop | claude_desktop_config.json に設定追加 | APIキー |
| Claude Code | settings.json に設定追加 | APIキー |
| コードから直接 | REST API | APIキー |

AIに不動産データを接続すると、「このエリアの相場は？」「投資利回りは？」といった質問に、実データに基づいた回答が返ってきます。

詳しい接続手順やプロンプト集は公式のMCP接続ガイドにまとめています。

https://fudosandb.jp/docs/mcp

---

*FUDOSAN DBは[カボシア株式会社](https://cabocia.jp)が運営しています。データソース: 国土交通省 不動産情報ライブラリ*
