# PQViewer 開発指示書

## 目的

このプロジェクトでは、ブラウザ上で動作する gnuplot-like な interactive plot viewer を開発する。  
将来的に以下のような機能を段階的に追加していく。

- 2D line / point / linepoints
- multiplot
- heatmap / colormap
- contour / contourf
- colorbar
- errorbar
- inset
- log scale
- annotation
- vector field
- 3D plot
- matplotlib 出力
- terminal / batch job からの figure 生成

開発においては、HTML viewer を単なる GUI として肥大化させるのではなく、**viewer config JSON を中心とする設計**を採用する。

---

## 基本方針

### 1. HTML viewer は preview と config 編集に集中する

HTML viewer の主な責務は以下とする。

- interactive preview
- UI による plot 設定編集
- data import
- config export / import
- PNG preview export
- matplotlib renderer 用 config export

HTML viewer は、可能な限り **描画仕様を JSON config として保持・出力する役割**に集中させる。

Python / matplotlib の詳細なコード生成ロジックを HTML 側に大量に持たせない。

---

### 2. Python renderer を公式な batch 出力 backend とする

terminal からの figure 生成は、HTML viewer が直接行うのではなく、Python 側の renderer が担当する。

想定コマンド例：

```bash
python plot2d_to_matplotlib.py \
  --config plot2d_config.json \
  --data data.csv \
  --out figure.png
```

PDF / SVG 出力も同じ renderer で扱う。

```bash
python plot2d_to_matplotlib.py \
  --config plot2d_config.json \
  --data data.csv \
  --out figure.pdf
```

---

### 3. 新しい描画機能は HTML / JSON schema / Python renderer をセットで設計する

Viewer に新しい描画機能を追加する場合、必ず次の 3 点を同時に検討する。

1. **HTML preview**
   - ブラウザ上でどう表示するか
   - どの UI で制御するか

2. **JSON config schema**
   - その機能をどのような JSON 構造で保存するか
   - 後方互換性をどう保つか

3. **Python / matplotlib renderer**
   - terminal からどのように再現するか
   - matplotlib のどの API に対応させるか
   - 完全再現が難しい場合は warning を出すか、近似するか

---

## 最重要設計原則

### JSON config を描画仕様の single source of truth とする

Viewer の状態は、最終的に JSON config として表現できるようにする。

HTML 内部状態、UI の値、Python renderer の挙動がばらばらにならないように、次を基本とする。

```text
UI state
  ↓
viewer config JSON
  ↓
HTML preview
  ↓
Python renderer
```

または、

```text
viewer config JSON
  ↓
HTML import
  ↓
HTML preview
```

```text
viewer config JSON
  ↓
Python renderer
  ↓
PNG / PDF / SVG
```

のように、JSON config を中心に据える。

---

## Config schema の基本構造

今後の拡張性を考慮して、config には必ず以下を含める。

```json
{
  "schema": "PQViewer",
  "version": 1,
  "data": {
    "included": false,
    "files": []
  },
  "layout": {},
  "plots": [],
  "styleConfig": {}
}
```

### `schema`

この config が本 viewer 用であることを示す。

```json
"schema": "PQViewer"
```

### `version`

config schema のバージョンを示す。

```json
"version": 1
```

将来的に構造を変更する場合は version を上げる。

Python 側では migration 関数を用意する。

```python
def migrate_config(config):
    """Migrate old config schema versions to the current schema."""
    version = config.get("version", 1)
    if version == 1:
        return migrate_v1_to_current(config)
    return config
```

Python コードを書く場合、コメントと docstring は英語で記述する。

---

## データの扱い

### 生データは原則として config に含めない

`export config` では、CSV / TSV / whitespace data の生データ配列を config に含めない。

理由：

- config が巨大化するのを避ける
- Git 管理しやすくする
- データと描画設定を分離する
- terminal / SLURM / WSL2 で再利用しやすくする

config には、生データではなく、データファイル参照と列指定を保存する。

例：

```json
{
  "data": {
    "included": false,
    "files": [
      {
        "id": "data1",
        "path": "data.csv",
        "format": "auto",
        "delimiter": "auto"
      }
    ]
  }
}
```

### series は列参照を持つ

line plot の場合、series は可能な限り次のように表現する。

```json
{
  "type": "line",
  "dataFileId": "data1",
  "xColumn": 1,
  "yColumn": 2,
  "legend": "sin(x)",
  "style": {
    "color": "#FE0000",
    "line": "solid",
    "width": 1.5,
    "point": "none",
    "pointSize": 3.2
  }
}
```

列番号は gnuplot 風に 1-based index を基本とする。  
Python 内部では必要に応じて 0-based に変換する。

---

## Plot item の抽象化

将来的な拡張を考え、plot 内の描画要素は `series` ではなく、可能なら `items` として抽象化する。

```json
{
  "plots": [
    {
      "id": "p1",
      "title": "2D Plot",
      "items": [
        {
          "type": "line",
          "dataFileId": "data1",
          "xColumn": 1,
          "yColumn": 2
        }
      ]
    }
  ]
}
```

### 推奨される item type

将来的に以下の type を追加できるようにする。

```text
line
scatter
linepoints
errorbar
heatmap
contour
contourf
image
vector
surface3d
annotation
fillbetween
```

type ごとに schema を分ける。

---

## Python renderer の設計

Python renderer は、type ごとの描画関数を dispatcher で呼び分ける設計にする。

```python
def render_line(ax, item, data_registry):
    """Render a line plot item on the given Matplotlib axes."""
    ...


def render_scatter(ax, item, data_registry):
    """Render a scatter plot item on the given Matplotlib axes."""
    ...


def render_contour(ax, item, data_registry):
    """Render a contour plot item on the given Matplotlib axes."""
    ...


RENDERERS = {
    "line": render_line,
    "scatter": render_scatter,
    "contour": render_contour,
}
```

描画時は以下のように dispatch する。

```python
def render_item(ax, item, data_registry):
    """Dispatch a plot item to the corresponding renderer."""
    kind = item.get("type", "line")
    renderer = RENDERERS.get(kind)
    if renderer is None:
        raise ValueError(f"Unsupported plot item type: {kind}")
    renderer(ax, item, data_registry)
```

新しい描画機能を追加する場合は、基本的に以下の手順にする。

1. config schema に `type` を追加
2. HTML viewer に UI と preview を追加
3. Python renderer に `render_xxx()` を追加
4. `RENDERERS` に登録
5. import / export / backward compatibility test を追加

---

## HTML 側の実装方針

### 既存 UI への変更は config との対応を明確にする

UI の control を追加する場合、以下を必ず確認する。

- `defaultStyle()` または default config に初期値を追加する
- `readControls()` で UI から state/config に反映する
- `writeControls()` で state/config から UI に反映する
- `exportCfg()` に反映される
- `importCfg()` 後に UI と preview が一致する
- Python renderer に必要な情報が config に含まれる

---

### UI だけの機能と描画機能を区別する

以下は HTML only の機能としてよい。

- panel collapse
- drag UI
- shortcut
- control panel layout
- help dialog
- color picker UI

以下は Python renderer にも反映が必要。

- line style
- point style
- axis range
- log scale
- title
- panel label
- legend
- grid
- margins
- font
- multiplot layout
- heatmap
- colormap
- contour
- colorbar
- errorbar
- annotations
- inset

---

## matplotlib 出力との対応方針

### 完全一致ではなく、再現性と論文用出力を優先する

HTML canvas と matplotlib では、以下が完全一致しない場合がある。

- font rendering
- dash pattern
- marker size
- text baseline
- legend box position
- margin calculation
- MathJax 表示
- canvas pixel coordinate と matplotlib axes coordinate の違い

そのため、Python renderer は **見た目の完全一致**ではなく、以下を重視する。

- 論文用 figure として自然な見た目
- 再現性
- terminal / batch での安定出力
- PDF / SVG 出力への対応
- config による一貫した制御

完全再現が難しい機能については、Python 側で warning を出す。

```python
warnings.warn(
    "Exact MathJax rendering is not supported in the Matplotlib backend. "
    "The label is rendered using Matplotlib mathtext instead."
)
```

---

## multiplot 追加時の方針

multiplot は config schema に明示的に layout を持たせる。

```json
{
  "layout": {
    "type": "multiplot",
    "rows": 2,
    "cols": 2,
    "sharex": false,
    "sharey": false,
    "hspace": 0.25,
    "wspace": 0.25
  },
  "plots": [
    {
      "id": "p1",
      "row": 0,
      "col": 0,
      "items": []
    },
    {
      "id": "p2",
      "row": 0,
      "col": 1,
      "items": []
    }
  ]
}
```

Python では `plt.subplots()` または `GridSpec` に対応させる。

```python
fig, axes = plt.subplots(
    nrows=rows,
    ncols=cols,
    sharex=sharex,
    sharey=sharey,
)
```

より柔軟な layout が必要になったら `GridSpec` を使う。

---

## heatmap / colormap 追加時の方針

heatmap は line series とは別 type とする。

```json
{
  "type": "heatmap",
  "dataFileId": "data1",
  "xColumn": 1,
  "yColumn": 2,
  "zColumn": 3,
  "gridMode": "auto",
  "colormap": "viridis",
  "colorbar": {
    "show": true,
    "label": "Intensity",
    "fontSize": 18
  }
}
```

Python 側では、データ形式に応じて以下を使い分ける。

- regular grid: `imshow`
- rectilinear grid: `pcolormesh`
- scattered data: `tricontourf` or interpolation

最初は regular / rectilinear grid を優先し、scattered data は後で対応してよい。

---

## contour / contourf 追加時の方針

contour は以下のように表現する。

```json
{
  "type": "contour",
  "dataFileId": "data1",
  "xColumn": 1,
  "yColumn": 2,
  "zColumn": 3,
  "levels": 20,
  "filled": false,
  "colormap": "viridis",
  "lineColor": "#000000",
  "lineWidth": 1.0,
  "colorbar": {
    "show": false
  }
}
```

Python 側では：

- `filled: false` → `ax.contour`
- `filled: true` → `ax.contourf`

に対応させる。

---

## Config export の方針

### 通常 config export

通常の `Export config` では、生データは含めない。

```javascript
function exportCfg() {
  const cfg = JSON.parse(JSON.stringify(S));
  // Remove raw data arrays.
  ...
}
```

ただし、series の style や data column reference は残す。

### 必要なら bundle export を別機能として作る

将来的に必要であれば、生データも含める export は別名にする。

```text
Export config
Export config with embedded data
Export render bundle
```

デフォルトでは生データを含めない。

---

## Import の方針

import 時には以下を行う。

1. schema / version を確認する
2. 古い config なら migrate する
3. 欠けている default 値を補う
4. UI に反映する
5. preview を再描画する
6. data が不足している場合は warning を出す

例：

```javascript
function normalizeImportedConfig(cfg) {
  cfg.schema = cfg.schema || 'PQViewer';
  cfg.version = cfg.version || 1;
  // Fill missing defaults here.
  return cfg;
}
```

---

## 後方互換性

古い config をできるだけ読み込めるようにする。

新しい property を追加する場合は、必ず fallback を用意する。

```javascript
s.titleShow = s.titleShow !== false;
s.legendSampleLen = s.legendSampleLen ?? 48;
s.legendPointSize = s.legendPointSize ?? 5;
```

Python 側でも同様に fallback を用意する。

```python
legend_sample_len = style.get("legendSampleLen", 48)
legend_point_size = style.get("legendPointSize", 5)
```

---

## テスト方針

新機能を追加したら、少なくとも以下を確認する。

### HTML viewer 側

- 起動時に console error が出ない
- Demo plot が表示される
- UI control が反映される
- config export できる
- config import できる
- export → import 後に見た目が大きく変わらない
- data が config に埋め込まれていない
- PNG export が動作する

### Python renderer 側

- config + data から PNG が生成できる
- PDF が生成できる
- SVG が生成できる
- line / point / linepoints が再現される
- legend / title / label / tick / font が反映される
- unsupported feature に対して明確な warning を出す

---

## AI に実装を依頼する際の指示テンプレート

以下を毎回 AI に渡す。

```text
この HTML viewer に機能を追加してください。

重要な設計方針:
- HTML viewer は interactive preview と config 編集に集中する。
- JSON config を描画仕様の single source of truth とする。
- terminal からの論文用 figure 出力は Python/matplotlib renderer が担当する。
- 新しい描画機能を追加する場合は、HTML UI / JSON schema / Python renderer の対応を必ず同時に考慮する。
- 生データは通常の export config には含めない。data file reference と column reference を config に残す。
- config には schema と version を持たせ、後方互換性を考慮する。
- Python renderer は item type ごとの renderer 関数と dispatcher で拡張する。
- HTML canvas と matplotlib の完全一致よりも、再現性・保守性・論文用出力を優先する。
- Python コードを書く場合、コメントと docstring は英語で書く。

今回追加したい機能:
[ここに機能を書く]

実装時に必ず確認すること:
1. default config / defaultStyle に初期値を追加する。
2. readControls / writeControls に対応を追加する。
3. export config / import config で設定が保持されるようにする。
4. その機能が最終図に影響する場合は、Python renderer 側の config schema と描画方針も提案する。
5. 古い config を読み込めるように fallback を用意する。
6. 生データを export config に混入させない。
7. 実装後、変更点とテスト項目を説明する。
```

---

## 新機能追加時のチェックリスト

AI は新機能を追加するとき、以下のチェックリストを満たすこと。

```text
[ ] この機能は HTML only か、最終図に反映すべき描画機能かを分類した。
[ ] config schema にどう保存するかを設計した。
[ ] default 値を定義した。
[ ] UI control を追加した。
[ ] readControls に反映した。
[ ] writeControls に反映した。
[ ] draw / preview に反映した。
[ ] export config に含まれることを確認した。
[ ] import config 後に復元されることを確認した。
[ ] 古い config 用の fallback を入れた。
[ ] Python renderer 側の対応方針を示した。
[ ] 必要なら render_xxx() の追加方針を示した。
[ ] unsupported / approximate な挙動があれば warning 方針を示した。
[ ] 生データを config に含めていない。
[ ] 動作確認項目を列挙した。
```

---

## 長期的な推奨構成

最終的には以下の構成を目指す。

```text
plot-viewer/
├── viewer/
│   └── PQViewer.html
├── renderer/
│   ├── plot2d_to_matplotlib.py
│   ├── renderers/
│   │   ├── line.py
│   │   ├── scatter.py
│   │   ├── heatmap.py
│   │   └── contour.py
│   └── schema.py
├── examples/
│   ├── data.csv
│   ├── config.json
│   └── figure.png
└── docs/
    └── config_schema.md
```

初期段階では単一 HTML と単一 Python script でよい。  
ただし、multiplot / heatmap / contour まで進む場合は、renderer 側を module 分割する。

---

## 最終方針

今後の開発では、以下を最優先する。

1. **config JSON 中心設計**
2. **HTML viewer と Python renderer の責務分離**
3. **生データと描画設定の分離**
4. **versioned schema**
5. **item type による拡張**
6. **dispatcher 型 Python renderer**
7. **後方互換性**
8. **terminal / batch / SLURM での再現性**
9. **論文用 figure として自然な matplotlib 出力**
10. **機能追加時に HTML / JSON / Python をセットで考える**

この方針に従い、今後の機能追加を設計・実装すること。
