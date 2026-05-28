# 講義スライド：インタラクティブWebGISで学ぶボロノイ図と地形解析

> **対象**: 情報系・プログラミング学習者（JavaScript 基礎知識あり、GIS 未経験可）  
> **所要時間**: 90 分（スライド 60 分 + デモ 20 分 + 演習 10 分）  
> **スライド枚数目安**: 35 枚  
> **デモ URL**: https://marumaruyama.github.io/volonoi-sample-Gemini/

---

## スライド構成（Google スライドへのコピー用）

各ブロックが 1 枚のスライドに対応しています。  
タイトル行 → スライドタイトル、箇条書き → 本文。

---

---
### [表紙]
# インタラクティブ WebGIS で学ぶ
# ボロノイ図と地形解析

**—— JavaScript × 地図 API × 国土地理院 DEM ——**

> 丸山友靖  
> ○○大学 情報学部  
> 2026 年

---

---
### [Agenda]
# 今日の内容

1. ボロノイ図ってなに？（概念と応用）
2. WebGIS の仕組み（ブラウザだけで地図を動かす）
3. ボロノイ図の計算（Delaunay 三角分割との関係）
4. 国土地理院 DEM タイルで地形を読む
5. システム実装のポイント（コードを見ながら）
6. デモ＆ハンズオン（自分で動かしてみよう）

---

---
## PART 1 ボロノイ図ってなに？

---
### [ボロノイ図の定義]
# ボロノイ図（Voronoi diagram）

**「最も近い施設はどこか？」を面で表した地図**

- 平面上に複数の点（施設）を置く
- 各点の「勢力圏」= 自分が最も近い領域
- 境界線はすべて隣接 2 点の垂直二等分線

```
        施設 A            施設 B
          ●                 ●
           \       境界      /
            \      ↓       /
   ← Aの     \────────────/     Bの →
      エリア   \          /     エリア
                ●
              施設 C
```

> 直感的な「なわばり地図」

---

---
### [生活の中のボロノイ]
# ボロノイ図が使われている場面

| 分野 | 例 |
|---|---|
| 都市計画 | 学区・医療圏・郵便局配置の分析 |
| 流通・小売 | 店舗の商圏分析、物流拠点の最適化 |
| 自然科学 | 降水量の内挿（ティーセン多角形） |
| ゲーム・CG | 地形生成、AI の行動範囲設定 |
| 生物学 | 細胞のなわばり・結晶構造の解析 |

> **今日作るもの**: 学校区・商圏を地図上でリアルタイムに可視化

---

---
### [ドロネー三角分割との双対性]
# 計算の核心：Delaunay 三角分割

ボロノイ図は **Delaunay 三角分割の双対グラフ**

```
  ドロネー三角形         ボロノイ図
  （各辺が入力点間）   （各辺が垂直二等分線）

      ●─────●               ┃
     /│\   /               ─┼─
    / │  ╲╱                 ┃
   ●──●──●           ─────●─────
    \ │  /╲                 ┃
     \│/   \               ─┼─
      ●─────●               ┃
```

1. 入力点の Delaunay 三角分割を計算（O(n log n)）
2. 各三角形の外接円中心を結ぶ → ボロノイ図

---

---
### [ライブラリ紹介: d3-delaunay]
# d3-delaunay を使う

```javascript
// 入力: ピクセル座標の配列 [[x1,y1], [x2,y2], ...]
const delaunay = d3.Delaunay.from(points);
const voronoi  = delaunay.voronoi([0, 0, width, height]);

// 各セルのポリゴン頂点を取得
for (let i = 0; i < points.length; i++) {
  const polygon = voronoi.cellPolygon(i);
  // polygon = [[x,y], [x,y], ...]
}
```

- **d3-delaunay** は Fortune のアルゴリズムを実装
- O(n log n) で高速動作
- CDN 1 行で使用可能

```html
<script src="https://cdn.jsdelivr.net/npm/d3@7/+esm"></script>
```

---

---
## PART 2 WebGIS の仕組み

---
### [Leaflet.js の役割]
# 地図ライブラリ：Leaflet.js

**「タイル地図 + 座標変換 + インタラクション」を担当**

```javascript
// 地図の初期化
const map = L.map('map').setView([35.68, 139.75], 12);

// OSM タイルを重ねる
L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);

// 重要：地理座標 → ピクセル座標の変換
const pt = map.latLngToLayerPoint([35.68, 139.75]);
// → SVG の x, y に使う
```

> Leaflet が座標変換を肩代わりしてくれるので
> 「地図上にどう描くか」に集中できる

---

---
### [SVG オーバーレイの仕組み]
# ボロノイ図を地図に重ねる

```
┌─────────────────────────────────┐
│  Leaflet 地図 (div#map)          │
│  ├── タイルレイヤー（PNG 画像）  │
│  └── SVG オーバーレイペイン      │
│       └── <polygon> × n個       │
│           ↑ ボロノイセル         │
└─────────────────────────────────┘
```

```javascript
// SVG を Leaflet の上に重ねる
const svg = d3.select(map.getPanes().overlayPane)
              .append('svg');

// 地図移動のたびに再描画
map.on('moveend zoomend', redrawVoronoi);
```

- パン・ズームするたびに「緯度経度 → ピクセル」を再計算
- 地図タイルと SVG が自動的に合わさる

---

---
### [全体アーキテクチャ]
# システム構成図

```
┌────────────────────────────────────────────┐
│            ブラウザ（クライアントのみ）       │
│                                             │
│  Leaflet.js   ←→   D3 / d3-delaunay        │
│  (地図・座標変換)    (ボロノイ計算・色分け)  │
│        ↑                    ↑              │
│        └─────── 操作 UI ────┘              │
│                      ↓                     │
│           傾斜解析エンジン                   │
└──────────────┬─────────────────────────────┘
               │ HTTP GET（非同期）
    ┌──────────┴───────────────┐
    │  国土地理院 DEM タイル    │
    │  cyberjapandata.gsi.go.jp │
    └──────────────────────────┘
```

> **サーバサイド不要** → GitHub Pages で無料公開可能

---

---
## PART 3 ボロノイ図の実装

---
### [座標変換フロー]
# 緯度経度 → SVG に描くまでの流れ

```
入力データ (JSON)
  { lat: 35.32, lng: 139.55, name: "○○小学校" }
       ↓  map.latLngToLayerPoint()
ピクセル座標
  { x: 412, y: 289 }
       ↓  d3.Delaunay.from(points)
Delaunay 三角分割
       ↓  delaunay.voronoi()
ボロノイポリゴン
  [[412,289], [387,301], ...]
       ↓  map.layerPointToLatLng() × 頂点数
地理座標に戻す（面積計算・クリップ用）
       ↓  SVG <polygon> に描画
```

---

---
### [セルの色分け実装]
# D3 カラースケールで色分け

```javascript
// 面積比モード：薄黄 → 濃赤
const areaScale = d3.scaleSequential(d3.interpolateYlOrRd)
                    .domain([0, 1]);

// 傾斜度モード：淡橙 → 濃橙
const slopeScale = d3.scaleSequential(d3.interpolateOranges)
                     .domain([0, 1]);

function cellFill(idx, opacity) {
  if (colorMode === 'area') {
    const ratio = areas[idx] / maxArea;
    return areaScale(ratio);
  }
  if (colorMode === 'slope') {
    const ratio = normalize(maxSlope[idx], allMaxSlopes);
    return slopeScale(ratio);
  }
  return `rgba(220, 50, 50, ${opacity})`;
}
```

---

---
## PART 4 国土地理院 DEM で地形を読む

---
### [DEM タイルとは]
# 国土地理院 DEM タイル

**DEM = Digital Elevation Model（数値標高モデル）**

```
URL パターン:
https://cyberjapandata.gsi.go.jp/xyz/{種別}/{z}/{x}/{y}.txt

種別の優先順位:
  dem5a → dem5b → dem
  （5m レーザー） （5m 写真） （10m）
```

- 1 タイル = テキスト形式の 256 × 256 標高値グリッド
- ズームレベル 14 で約 2.4km × 2.4km / タイル
- **無料・CC BY 4.0** → 商用利用も可

```javascript
const url = `https://cyberjapandata.gsi.go.jp/xyz/dem5a/14/${x}/${y}.txt`;
const res  = await fetch(url);
const text = await res.text();
// "12.34,13.56,14.01,..." をパース
```

---

---
### [タイル座標の計算]
# 緯度経度 → タイル座標 (x, y)

```javascript
function latlng2tile(lat, lng, zoom) {
  const n    = Math.pow(2, zoom);
  const tileX = Math.floor((lng + 180) / 360 * n);
  const latRad = lat * Math.PI / 180;
  const tileY = Math.floor(
    (1 - Math.log(Math.tan(latRad) + 1 / Math.cos(latRad)) / Math.PI)
    / 2 * n
  );
  return { x: tileX, y: tileY };
}
// 例: 鎌倉 (35.32, 139.55) zoom=14 → { x: 14552, y: 6473 }
```

タイル内ピクセル位置 (px, py) も計算 → 標高値の行・列インデックスに変換

---

---
### [8×8 サンプリンググリッド]
# 面内のサンプル点配置

各ボロノイセルの外接矩形に **8×8=64点** を格子状に配置

```
┌─────────────────────────────┐
│ ·   ·   ·   ·   ·   ·   · │
│ ·   ●   ●   ●   ●   ●   · │   ● セル内部の点
│ ·   ●   ●   ●   ●   ●   · │       → 標高を取得
│ ·   ·   ●   ●   ●   ●   · │   · セル外部の点
│ ·   ·   ·   ●   ●   ●   · │       → スキップ
│ ·   ·   ·   ·   ·   ·   · │
└─────────────────────────────┘
```

**内外判定 = レイキャスト法**（Jordan curve theorem）

```javascript
function pointInPolygon(point, polygon) {
  let inside = false;
  for (let i = 0, j = polygon.length - 1; i < polygon.length; j = i++) {
    const xi = polygon[i][0], yi = polygon[i][1];
    const xj = polygon[j][0], yj = polygon[j][1];
    if ((yi > point[1]) !== (yj > point[1]) &&
        point[0] < (xj - xi) * (point[1] - yi) / (yj - yi) + xi)
      inside = !inside;
  }
  return inside;
}
```

---

---
### [傾斜角の計算]
# 傾斜角 θ の算出式

隣接サンプル点ペアから傾斜角を計算

```
    ↑ 標高 (m)
    │
 hB ┤ ○ 点 B
    │  ╲
    │   ╲ Δh = |hB − hA|
    │    ╲
 hA ┤─────○──────────────→ 水平距離 (m)
   点A        d
```

```javascript
// θ = arctan( Δh / d ) [度]
function slopeAngle(h1, h2, lat1, lng1, lat2, lng2) {
  const dh = Math.abs(h2 - h1);
  const d  = haversine(lat1, lng1, lat2, lng2); // 球面距離 [m]
  return Math.atan2(dh, d) * 180 / Math.PI;
}
```

> **ハバーサイン公式**で球面上の正確な距離を計算

---

---
### [3つの傾斜指標]
# 1回のDEM取得で3指標を算出

```javascript
async function calcSlopeStats(polygon) {
  const samples = getSamplePoints(polygon); // 8×8 グリッド
  const heights = await fetchAllElevations(samples); // DEM 取得

  const angles = [];
  for (/* 右方向・下方向の隣接ペア */) {
    angles.push(slopeAngle(h1, h2, ...));
  }

  return {
    max:  Math.max(...angles),          // 最大傾斜度
    avg:  mean(angles),                 // 平均傾斜度
    diff: Math.max(...angles) - mean(angles)  // 最大−平均差
  };
}
```

| 指標 | 意味 |
|---|---|
| 最大傾斜度 | 面内で最も急な場所 → 崖・急斜面の検出 |
| 平均傾斜度 | 面全体の傾斜水準 → 開発適性の評価 |
| 最大−平均差 | 地形の**不均一性** → 谷戸型地形の検出 |

---

---
### [キャッシュ戦略]
# DEM タイルのキャッシュ

1 タイルに複数のサンプル点が含まれる → 重複取得を避ける

```javascript
const demCache = {}; // { "14/14552/6473": Float32Array }

async function fetchElevation(lat, lng) {
  const { x, y } = latlng2tile(lat, lng, 14);
  const key = `14/${x}/${y}`;

  if (!demCache[key]) {
    const res  = await fetch(GSI_URL(x, y));
    demCache[key] = parseDEM(await res.text());
  }

  const { px, py } = pixelInTile(lat, lng, x, y);
  return demCache[key][py * 256 + px];
}
```

> リクエスト数を大幅削減 → 体感速度が向上

---

---
## PART 5 事例：鎌倉市立小学校の通学圏分析

---
### [鎌倉を選んだ理由]
# なぜ鎌倉市？

**「同じ市内で地形が全然違う」**

```
    北 ←──── 鎌倉市 ────→ 南（相模湾）

  大船・玉縄       鎌倉中心部     七里ガ浜
  （丘陵・台地）   （谷戸地形）    （沿岸平野）
    ↑傾斜大           ↑急崖点在       ↑ほぼ平坦
```

- 市内に小学校 **12 校** → ボロノイの比較に最適
- 海岸・谷戸・丘陵が近距離で混在
- 5m DEM（dem5a）が整備済みで高精度

---

---
### [分析結果]
# 結果：サービス圏と地形指標

| 校名 | 面積 | 最大傾斜 | 平均傾斜 | 最大−平均差 | 地形タイプ |
|---|---|---|---|---|---|
| 大船小学校 | 大 | 低 | 低 | 小 | 平坦 |
| 七里ガ浜 | 大 | 低 | 低 | 小 | 沿岸平野 |
| 玉縄小学校 | 大 | 中 | 中 | 中 | 一様丘陵 |
| 第三小学校 | 中 | **高** | 高 | 中 | 丘陵 |
| 山崎小学校 | 中 | **高** | 中 | **大** | 谷戸混在 |
| 笛田小学校 | 小 | 中 | 低 | **大** | 谷戸型 |

**注目**: 笛田・山崎は「最大は高いが平均は低い」→ 差が大きい = 谷戸！

---

---
### [谷戸地形の可視化]
# 最大−平均差（Δθ）の意義

```
  差が大きい（谷戸型）          差が小さい（一様斜面型）

  急│   ╱╲                    急│
    │  ╱  ╲                    │  ╱╱╱╱╱
    │──    ──                緩│ ╱╱╱╱╱╱
  緩│                            └────────────→
    └────────────→
    θmax 高・θavg 低             θmax 中・θavg 中
    Δθ: 大 → 濃橙                Δθ: 小 → 淡橙
```

- 最大値だけ見ると「どこも同じくらい急」に見えてしまう
- Δθ（差）を見ると「急斜面が局所的か？広域的か？」が分かる
- → 谷戸地形・崖地の自動識別に応用可能

---

---
## PART 6 実装のポイント（コードツアー）

---
### [主要ライブラリ]
# 使用ライブラリ一覧

```html
<!-- 地図表示 -->
<script src="leaflet.js"></script>
<link  rel="stylesheet" href="leaflet.css">

<!-- ボロノイ計算・色スケール -->
<script src="https://cdn.jsdelivr.net/npm/d3@7/+esm" type="module"></script>

<!-- Google Maps オプション -->
<script src="Leaflet.GoogleMutant.js"></script>
```

| ライブラリ | 役割 | 特徴 |
|---|---|---|
| Leaflet.js v1.9 | 地図表示・操作 | 軽量・モバイル対応 |
| D3.js v7 | ボロノイ・色分け | データ可視化の定番 |
| d3-delaunay v6 | 高速ボロノイ計算 | Fortune's algorithm |
| Leaflet.GoogleMutant | Google Maps 連携 | タイル無断利用なし |

---

---
### [Google Maps API の組み込み]
# GitHub Actions でシークレット注入

API キーをソースに直書きしない → GitHub Secrets を使う

```yaml
# .github/workflows/deploy.yml
- name: Inject API key
  run: sed -i "s/__GOOGLE_MAPS_API_KEY__/${{ secrets.MAPS_API_KEY }}/g" index.html

- name: Deploy to GitHub Pages
  uses: peaceiris/actions-gh-pages@v3
```

```javascript
// index.html（プレースホルダー）
const GOOGLE_API_KEY = "__GOOGLE_MAPS_API_KEY__";
```

> デプロイ時だけキーが埋め込まれる → リポジトリには残らない

---

---
### [モバイル対応]
# レスポンシブ設計のポイント

```css
/* PC: サイドバーは常時表示 */
#sidebar { width: 240px; }

/* モバイル (640px 以下): ドロワー式 */
@media (max-width: 640px) {
  #sidebar {
    position: fixed;
    transform: translateX(-100%);    /* 初期: 画面外 */
    transition: transform 0.3s;
  }
  #sidebar.open {
    transform: translateX(0);        /* 開くとき */
  }
  #map { height: 100dvh; }          /* dvh = Dynamic Viewport Height */
}
```

- `dvh` = ブラウザのアドレスバーを考慮した高さ（スマホで必須）
- ハンバーガーメニューでサイドバーをトグル

---

---
## PART 7 デモ＆演習

---
### [デモ手順]
# デモ：触ってみよう

**URL**: https://marumaruyama.github.io/volonoi-sample-Gemini/

1. **JSON 読み込み**  
   → Import JSON から `kamakura_elementary_schools.json` を選択

2. **色分けを切り替える**  
   → 「面積比」→「最大傾斜度」→「最大−平均傾斜度」と順番に

3. **背景を変える**  
   → 「Google 衛星画像」で地形を確認しながら比較

4. **店舗を追加・削除する**  
   → 地図クリックで追加 / Shift+クリックまたは🗑️ボタンで削除

---

---
### [演習課題]
# 演習：自分のデータを作ってみよう

**課題**: 自分の街の施設データを JSON で作り、ボロノイ図を描く

```json
[
  { "id": 1, "name": "駅A", "lat": 35.XXX, "lng": 139.XXX },
  { "id": 2, "name": "駅B", "lat": 35.XXX, "lng": 139.XXX }
]
```

**座標の調べ方**:
1. Google マップで場所を右クリック
2. 表示された数字（例: 35.6895, 139.6917）をコピー

**アイデア**:
- コンビニ / スーパー / 図書館の分布
- 最寄り駅のサービス圏
- 病院・クリニックの配置

> JSON ファイルをブラウザにドラッグ＆ドロップするだけ！

---

---
### [発展課題]
# もっとやりたい人へ

**① コードを読む**  
GitHub: https://github.com/MaruMaruyama/volonoi-sample-Gemini  
`index.html` 1 ファイルに全ロジックが入っている

**② 機能追加アイデア**
- 人口データとの重ね合わせ（e-Stat API）
- 道路距離ベースの等時圏分析（Mapbox / OSRM）
- ポイント数が変わったときのアニメーション

**③ 傾斜アルゴリズムを改良**
- 8×8 グリッド → より高密度なサンプリング
- Horn の方法（8 近傍差分）による勾配ベクトル計算

---

---
### [まとめ]
# まとめ

| テーマ | 学んだこと |
|---|---|
| ボロノイ図 | 最近傍施設の「なわばり」を面で表す |
| WebGIS | Leaflet × D3 でブラウザ完結の地図アプリ |
| DEM 解析 | 国土地理院タイルで傾斜を計算 |
| 3 傾斜指標 | 最大・平均・差で地形の「性格」を把握 |
| 実装のコツ | 座標変換・キャッシュ・SVG オーバーレイ |

**このシステムは全て無料のオープンデータと OSS で構築**  
→ 今日から誰でも同じものが作れる！

---

---
### [参考資料]
# 参考資料

- **デモ**: https://marumaruyama.github.io/volonoi-sample-Gemini/
- **コード**: https://github.com/MaruMaruyama/volonoi-sample-Gemini
- Leaflet.js: https://leafletjs.com/
- d3-delaunay: https://github.com/d3/d3-delaunay
- 国土地理院 DEM: https://maps.gsi.go.jp/development/ichiran.html
- Okabe et al. (2000) *Spatial Tessellations*. Wiley.

---

---

## スライド作成メモ（Google スライドへのコピー手順）

1. 各 `---` ブロック = 1 枚のスライド
2. `###` タイトル → スライドのタイトルテキストボックス
3. コードブロック → 「等幅フォント（Source Code Pro / Courier）」のテキストボックス、背景色 #1e1e1e（白文字）
4. ASCII 図 → 等幅フォントのテキストボックスでそのままペースト
5. 表 → Google スライドの「表の挿入」で再現
6. 推奨テーマ: **Simple Dark** または **Momentum**（暗背景がコードを映える）
