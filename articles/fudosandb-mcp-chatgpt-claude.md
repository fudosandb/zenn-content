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

FUDOSAN DBのMCP Serverを接続するだけで、AIが国交省の不動産データにアクセスできるようになります。取引価格730万件、地価公示、賃料推定AIまで。

## FUDOSAN DBとは

国交省の不動産情報ライブラリ（reinfolib）のデータを構造化して提供するAPIサービスです。

reinfolib APIを直接使うと、PBFデコード、座標変換、データ正規化といった前処理に数週間かかります。FUDOSAN DBはこれらの前処理済みデータを、RESTful JSONとMCPの両方で提供しています。

- 取引価格データ 730万件
- 地価公示・地価調査
- 賃料推定AI（誤差2%台）
- エリア分析・ランキング・スクリーナー
- 申請不要、APIキー即時発行、無料プランあり

https://fudosandb.jp

## MCPで接続する（Claude Desktop）

### 1. claude_desktop_config.jsonに追加

```json
{
  "mcpServers": {
    "fudosandb": {
      "url": "https://fudosandb.jp/mcp"
    }
  }
}
```

これだけです。APIキーなしでも基本機能が使えます。

### 2. 使ってみる

Claude Desktopで以下のように聞いてみてください。

**エリアの相場を調べる:**
> 「港区と世田谷区の不動産取引価格を比較して」

**賃料を推定する:**
> 「渋谷区で25m²、築10年の1Kの家賃を推定して」

**地価トレンドを見る:**
> 「千代田区の過去5年の地価推移を教えて」

**条件でエリアを絞り込む:**
> 「東京23区で地価上昇率が高い順にランキングして」

MCPツールが用意されているので、これらの質問に対してAIが適切なツールを選んでデータを取得・整理してくれます。

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

MCPを使わず、REST APIとして利用することもできます。

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
| MCP対応 | α版（25/35 API） | 対応済み |
| 賃料推定 | なし | あり（誤差2%台） |
| 料金 | 無料 | 無料プランあり |

reinfolib APIは生データを自分で加工したい場合に向いています。FUDOSAN DBは構造化済みのデータをすぐ使いたい、AIから接続したいという場合に向いています。用途に応じて使い分けてください。

## まとめ

MCP Serverの設定は1行。AIに不動産データを接続すると、「このエリアの相場は？」「投資利回りは？」といった質問に、実データに基づいた回答が返ってきます。

https://fudosandb.jp

---

*FUDOSAN DBは[カボシア株式会社](https://cabocia.jp)が運営しています。データソース: 国土交通省 不動産情報ライブラリ*
