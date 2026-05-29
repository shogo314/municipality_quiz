# 市区町村クイズ 開発メモ (README)

`municipality_quiz.html` 1ファイルで完結する、市区町村名から都道府県を地図クリックで当てるクイズです。ビルド不要・通信不要で、ブラウザで開けば動きます。このドキュメントは後から自分で改造するための設定・データ構造・関数の説明です。

---

## 1. ファイル構成

HTML 1枚の中に「データ（JSON）」「スタイル（CSS）」「ロジック（JS）」が同居しています。

```
municipality_quiz.html
├─ <style> … 見た目（CSS変数で色・フォントを集中管理）
├─ DOM     … 出題パネル + 地図コンテナ
├─ <script type="application/json" id="geo">  … 47都道府県の境界データ(GeoJSON)
├─ <script type="application/json" id="data"> … 市区町村データ(munis/nameToPrefs)
└─ <script> … 本体ロジック
```

`id="geo"` と `id="data"` の2つは「実行されないJSON」です。本体スクリプトが起動時に `JSON.parse` で読み込みます。

開発用の元ファイル（手元に保管推奨）:

- `template2.html` … データ部分が `__GEO__` / `__DATA__` というプレースホルダになったテンプレート
- `japan_simplified.geojson` … 簡略化済みの都道府県境界（28KB）
- `munis.json` … 市区町村データ
- `000925835.xlsx` … 元データ（総務省 全国地方公共団体コード R6.1.1）

最終ファイルは「テンプレートのプレースホルダを実データで置換」して生成します（後述§6）。`municipality_quiz.html` を直接編集してもよいですが、JSONが巨大なので**ロジックや見た目の変更は `template2.html` 側で行い、再ビルドする**のが楽です。

---

## 2. データソースと出典

- 出典: 総務省「全国地方公共団体コード」令和6年1月1日版（`000925835.xlsx`）
- 地図形状: 都道府県境界の公開GeoJSONを `mapshaper` で約1.2%に簡略化したもの。各featureの `properties.nam_ja` に「北海道」「東京都」等のフル名称が入っています（都道府県名はxlsxと47件完全一致）
- このxlsxには**緯度経度が無い**ため、地図形状は別途GeoJSONを使っています

---

## 3. すぐ変えたくなる設定（チューニング箇所）

### 3-1. 色・フォント（CSSの `:root`）

`<style>` 冒頭の CSS 変数を変えるだけで全体の配色が変わります。

| 変数 | 用途 |
|---|---|
| `--paper`, `--paper2` | 背景の紙色 |
| `--ink`, `--ink-soft` | 文字色 |
| `--land`, `--land-stroke`, `--sea` | 地図の陸・県境・海 |
| `--vermilion`, `--indigo` | アクセント（朱・藍） |
| `--ok`, `--ng`, `--hint` | 正解=緑 / 不正解=赤 / ヒント=黄 |

フォントは先頭の `@import`（Google Fonts）と、見出し用 `Shippori Mincho B1` / 本文用 `Zen Kaku Gothic New`。差し替えたい場合はここを変更します。

### 3-2. 地図サイズ・沖縄インセット

本体スクリプト上部:

```js
const W=600, H=660;          // 地図SVGのviewBox（描画は親幅に自動フィット）
const inW=120,inH=120,inX=10,inY=10;  // 沖縄インセット枠の 幅・高さ・左上座標
```

- 沖縄の枠を動かしたい → `inX`,`inY` を変更（例: 右上なら `inX = W-inW-10`）。現在は左上（鹿児島との重なり回避のため）
- 枠の大きさ → `inW`,`inH`
- ラベル「沖縄」の位置は `il.setAttribute('x', inX+6)` / `('y', inY+14)` の箇所

### 3-3. 出題範囲のプルダウン

HTML側 `<select id="range">` の選択肢と、JSの `buildPool()` の判定が対応しています。件数表示（792/932など）は表示上の文言なので、データを差し替えたら手で直してください。

### 3-4. ヒントのブロック分け（重要）

```js
const HINT_ZONES = { '北日本':[...7県], '関東':[...], ... };
const HINT_LABEL = { '北日本':'北日本（北海道・東北）', ... };
const PREF2HZONE = { 都道府県名 → ブロック名 };  // 上から自動生成
```

ヒントは**全問この6ブロック単位で統一**しています。これは「北海道だけ狭い範囲が光る＝答えがバレる」のを防ぐためで、北海道は東北と同じ「北日本」に固定。粒度を変えたい（例: 中国と四国を分ける、関東と中部を統合）場合は `HINT_ZONES` と `HINT_LABEL` を編集してください。**47県すべてがどこか1ブロックに属する**ようにすれば破綻しません。

---

## 4. データ構造

### 4-1. 埋め込みJSON

```js
DATA = {
  munis: [ { pref:"北海道", name:"札幌市", kana:"ｻｯﾎﾟﾛｼ" }, ... ],  // 1747件
  nameToPrefs: { "池田町":["北海道","岐阜県","長野県","福井県"], ... } // 同名(漢字)が複数県にある25件のみ
}
GEO = { type:"FeatureCollection", features:[ {properties:{nam_ja,id}, geometry:{...}}, ... ] } // 47件
```

※ `nameToPrefs` は参考情報で、現在のロジックは下記 `byName` を起動時に再構築して使っています。

### 4-2. 起動時に作る索引 `byName`

`DATA.munis` を**漢字名でグルーピング**したもの。クイズの出題単位です。

```js
byName = {
  "池田町": {
    instances: [ {pref:"北海道",kana:"イケダチョウ"}, {pref:"長野県",kana:"イケダマチ"}, ... ],
    prefs:     ["北海道","岐阜県","長野県","福井県"],  // 正解として要求する県（重複除去）
    diffRead:  true   // 同じ漢字で読みが複数あるか（"読み違い注意"バッジ用）
  }, ...
}
allNames = byName のキー一覧（出題プール元）
```

- `kana` は `toFull()`（`String.normalize('NFKC')`）で半角→全角カナに変換済み
- **「全県クリック必須」の根拠がこの `prefs`**。同名で複数県にまたがる25件は、その全県を当てて初めて正解になります

---

## 5. 関数・変数リファレンス

### 5-1. 共通ユーティリティ

| 名前 | 説明 |
|---|---|
| `$(id)` | `document.getElementById` の短縮 |
| `toFull(s)` | 半角カナ等を全角へ正規化（NFKC） |

### 5-2. 地図描画

| 関数 | 説明 |
|---|---|
| `polysOf(f)` | feature の geometry を「ポリゴン配列」に正規化（Polygon/MultiPolygon両対応） |
| `bboxOf(features)` | 経度緯度の最小外接矩形 `{minx,miny,maxx,maxy}` を返す |
| `makeProj(bb,w,h,padx,pady)` | 経緯度→SVG座標の射影関数を生成。緯度による横方向の縮み（`cos(平均緯度)`）を補正した簡易正距円筒図法。`w,h` の枠に収まるよう自動スケール＆中央寄せ |
| `pathD(f,proj)` | feature を SVG の `path` の `d` 文字列に変換 |
| `addPath(f,proj)` | path要素を生成し、クリック/ホバーのイベントを付けて SVG に追加。`pathByPref[県名]=[要素…]` に登録 |

関連変数:

- `mainF` … 沖縄以外の46地物、`okiF` … 沖縄
- `mainProj` … 本土用の射影、`okiProj` … 左上インセット用の射影
- `pathByPref` … `{ 県名: [path要素…] }`。県をまたぐ多角形もまとめて色変更できる
- `OKI = '沖縄県'`, `SVGNS` … SVG名前空間

### 5-3. ツールチップ

`showTip(e,name)` / `hideTip()` … ホバー中の県名を表示/非表示。`clearClasses()` … 全県の `cand/found/ng` クラスを除去（出題リセット用）。

### 5-4. クイズ状態（state）

```js
let pool=[];        // 現在の出題候補（名前の配列）
let cur=null;       // 出題中の市区町村名（byName のキー）
let found=new Set();// この問題で既に当てた県
let locked=true;    // 回答受付ロック（解決後はクリック無効）
let hinted=false;   // この問題でヒントを使ったか
let score,total,streak,best;  // 成績
const missed=new Set();        // 間違えた/降参した問題（復習用）
let reviewMode=false;          // 復習モード中か
```

### 5-5. クイズ進行

| 関数 | 説明 |
|---|---|
| `buildPool()` | `<select id="range">` の値で `allNames` を絞り `pool` を作る |
| `next()` | 次の問題を出す。状態リセット → ランダム抽選 → 表示更新。**多答問題（`prefs.length>1`）は読みを隠し、進捗「0/N」を表示**。単答は読みを表示 |
| `onClick(pref)` | 県クリックの判定。正解県なら `found` に追加して緑。**全県そろえば** `resolve('clear')`。違う県を1つでも押したら即 `resolve('wrong')` |
| `resolve(kind)` | 問題を確定。`kind`=`'clear'`(全部正解)/`'wrong'`(誤クリック)/`'giveup'`(降参)。正解県を全部開示し、各県の読み一覧を表示。スコア・連続・復習リストを更新 |
| `doHint()` | 答えを含む**広域ブロック**を黄色で表示＋読みも表示。`hinted=true` にし、以後その問題は正解にカウントしない |
| `updateStats()` | 数値表示と「復習(n)」ボタンの活性を更新 |
| `enterReview()` / `exitReview()` | 復習モードの開始/終了。復習中は `missed` から出題し、正解すると `missed` から消える |

採点ルールの要点（`resolve` 内）:

- `clear` かつ `hinted=false` → **正解**（score++, streak++, best更新, missedから除外）
- `clear` かつ `hinted=true` → クリア扱いだが**正解にカウントしない**（練習）
- `wrong` / `giveup` → **不正解**（streak=0, missedに追加）

### 5-6. イベント

- `#next` クリック / `Enter`キー → `next()`
- `#hint` クリック / `h`キー → `doHint()`
- `#giveup` → `resolve('giveup')`
- `#range` 変更 → 復習中なら解除 → `buildPool()` → `next()`
- `#review` / `#exitReview` → 復習モード切替

---

## 6. データや地図を差し替えて再ビルドする

最終HTMLは「テンプレート＋データ」で生成しています。ロジックや見た目は `template2.html` を編集し、以下で再ビルドします（Python標準ライブラリのみ）。

```python
geo  = open('japan_simplified.geojson').read().strip()
data = open('munis.json').read().strip()
html = open('template2.html').read().replace('__GEO__', geo).replace('__DATA__', data)
open('municipality_quiz.html','w').write(html)
```

### データ（munis.json）を作り直す場合

`000925835.xlsx`（シート `R6.1.1現在の団体`）から、都道府県名が入っていて市区町村名もある行だけ抽出します。

```python
import openpyxl, json
from collections import defaultdict
wb = openpyxl.load_workbook('000925835.xlsx', read_only=True)
ws = wb['R6.1.1現在の団体']
rows = list(ws.iter_rows(values_only=True))[1:]   # 1行目はヘッダ
data = [{'pref':r[1].strip(),'name':r[2].strip(),'kana':(r[4] or '').strip()}
        for r in rows if r[1] and r[2]]            # 列: 0=団体コード 1=都道府県 2=市区町村 4=市区町村カナ
n2p = defaultdict(set)
for d in data: n2p[d['name']].add(d['pref'])
out = {'munis':data, 'nameToPrefs':{n:sorted(p) for n,p in n2p.items() if len(p)>1}}
json.dump(out, open('munis.json','w'), ensure_ascii=False)
```

新年度版のコード表に差し替えるときも、列の並びが同じならこのまま使えます。

### 地図（GeoJSON）を作り直す/精度を変える場合

元の都道府県GeoJSON（`properties.nam_ja` を持つもの）を `mapshaper` で簡略化します。数値を上げると精細・重く、下げると粗・軽くなります。

```bash
mapshaper input.geojson -simplify 1.2% keep-shapes \
  -filter-fields nam_ja,id -o precision=0.001 japan_simplified.geojson
```

`nam_ja` の県名が `DATA` 側の `pref` と一致していることが前提です（不一致だとクリック判定が当たりません）。

---

## 7. よくある改造レシピ

- **誤クリックを即不正解にせず減点に変えたい** → `onClick` の `else` 節で `resolve('wrong')` する代わりに、ミス回数を数えて続行し、`found` が揃った時点で「ミスあり＝不正解扱い」にする
- **4択モードを足したい** → `next()` で `cur` の正解県＋ダミー3県のボタンを別UIに出し、判定だけ `byName[cur].prefs` と突き合わせる
- **市区町村そのものをクリックさせたい** → 都道府県GeoJSONを「市区町村境界GeoJSON（国土数値情報 行政区域データ等）」に差し替え、`nam_ja` 相当の市区町村名で `pathByPref` を作る（約1,747ポリゴンになるので簡略化必須）
- **成績を保存したい** → このプレビュー環境では `localStorage` が使えません。自分のサーバ/ローカルに置けば `localStorage.setItem('quizStats', JSON.stringify({score,total,best}))` で保存可能
- **タイマー/タイムアタック** → `next()` で開始時刻を記録、`resolve()` で経過を集計して表示

---

## 8. 既知の制限

- 地図は簡略化（約1.2%）しているため海岸線は粗い。クリック判定には十分
- 政令市の行政区（例: 札幌市中央区）は出題に含めていません（xlsx 2枚目シートにあるが未使用）
- ランダム出題は重複回避をしていない（直前と同じ問題が稀に連続し得る）
- 成績はページを閉じると消える（§7 参照）
