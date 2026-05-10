# Interactive Voronoi Market Simulator

このプロジェクトは、GIS（地理情報システム）の基本的な概念である**ボロノイ分割**を用いた、インタラクティブな商圏分析シミュレーターです。地図上での店舗配置・撤退が、周囲のナワバリ（商圏）にどのような影響を与えるかをリアルタイムに可視化します。

## デモ
https://marumaruyama.github.io/volonoi-sample-Gemini/index.html

## 主な機能

### 背景地図
| スタイル | 内容 |
|---|---|
| OSM（カラー） | OpenStreetMap（デフォルト） |
| OSM（グレースケール） | CSSフィルターによる白黒表示 |
| Google ロードマップ | Google Maps 道路地図 |
| Google ロードマップ（グレー） | Google Maps グレースケールスタイル |
| Google 衛星画像 | Google Maps 航空写真 |

Google Maps の表示には Google Maps API キーが必要です（後述）。

### ボロノイ面の色分けモード
| モード | 内容 |
|---|---|
| なし | 薄い赤で均一表示 |
| 面積比 | 面積に応じて黄→赤のグラデーション |
| 最大傾斜度（国土地理院DEM） | 各面内の最大傾斜角に応じて緑（緩）→赤（急）でグラデーション |

- **透過度スライダー**: ボロノイ面の不透明度を 5〜90% で調整可能
- **カラー凡例**: 色分けモード選択時に最小・最大値付きで凡例を表示

### 傾斜度算出（国土地理院DEMタイル）
最大傾斜度モードを選択すると、各ボロノイ面について以下の処理を実行します：

1. 面の境界ボックス内に 8×8 = 64 のサンプル点を配置
2. 各点が面内部にあるかレイキャスト法で判定
3. 国土地理院の標高タイル（`dem5a` → `dem5b` → `dem` の順でフォールバック）から標高値を取得
4. 隣接点間の標高差と水平距離から傾斜角（度）を算出し、最大値を採用
5. 取得済みタイルはキャッシュし、再取得を防止

使用する標高タイルURL（ズームレベル14）:
```
https://cyberjapandata.gsi.go.jp/xyz/dem5a/{z}/{x}/{y}.txt
```

### 基本操作
- **左クリック**: 店舗（赤ドット）を追加
- **Shift + 左クリック**: 店舗を削除
- **JSONインポート**: `{ lat, lng, id, name }` 形式の JSON ファイルから店舗データを読み込み

## 使用技術
- **[Leaflet.js](https://leafletjs.com/) v1.9.4**: インタラクティブな地図表示と座標変換
- **[D3.js](https://d3js.org/) v7 + [d3-delaunay](https://github.com/d3/d3-delaunay) v6**: ボロノイ図計算・色スケール
- **[Leaflet.GoogleMutant](https://github.com/IvanSanchez/Leaflet.GoogleMutant) v0.13.5**: Google Maps タイル統合
- **OpenStreetMap**: デフォルト背景地図
- **国土地理院 標高タイル**: 傾斜度算出用 DEM データ

## セットアップ

### Google Maps API キーの設定
Google Maps を使用するには、Google Cloud Console で Maps JavaScript API を有効にした API キーが必要です。

1. リポジトリの **Settings → Secrets and variables → Actions** を開く
2. `MAPS_API_KEY` という名前でシークレットを作成し、APIキーを設定する
3. `main` ブランチまたは `claude/voronoi-google-maps-rWqgR` ブランチにプッシュすると GitHub Actions が自動的にキーを埋め込んでデプロイします

### デプロイ（GitHub Pages）
```yaml
# .github/workflows/deploy.yml がトリガーするブランチ
branches:
  - main
  - claude/voronoi-google-maps-rWqgR
```
プッシュ時に GitHub Actions が `index.html` 内のプレースホルダー `__GOOGLE_MAPS_API_KEY__` を `MAPS_API_KEY` シークレットで置換した上で、GitHub Pages へ自動デプロイします。

## JSONデータ形式
```json
[
  { "id": 101, "name": "店舗A", "lat": 35.6597, "lng": 139.7003 },
  { "id": 102, "name": "店舗B", "lat": 35.6605, "lng": 139.6989 }
]
```

## 今後の展望
- 道路ネットワークを考慮した到達圏分析
- 人口統計データとの重ね合わせによる潜在顧客数推計
- 傾斜度に加えた土地利用データ等との複合分析
