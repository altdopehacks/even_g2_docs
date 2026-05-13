# UIデザインガイド

G2グラスの576×288pxキャンバスを使ったUI構築のガイドライン。

## キャンバス仕様

| 項目 | 値 |
|---|---|
| 解像度 | 576 × 288 px |
| 原点 | 左上 (0, 0) |
| X軸 | 右方向に増加（0〜576） |
| Y軸 | 下方向に増加（0〜288） |
| カラー深度 | 4ビットグレースケール（16段階の緑） |
| 白ピクセル | ハードウェア上では明るい緑として表示 |
| 黒ピクセル | オフ（透過）— 実世界が透けて見える |
| 背景色 | なし（未描画領域は透過） |

## コンテナのルール

ページはコンテナで構成されます。

- **合計最大12コンテナ**（全種類合計）
- **テキスト/リストコンテナは最大8個**
- **画像コンテナは最大4個**
- **`isEventCapture: 1` を持つコンテナはちょうど1つ**
- **z-indexなし** — 宣言順で重なり順が決まる（後に宣言したものが上）
- **`containerID`** はページ内で一意の数値
- **`containerName`** はページ内で一意（最大16文字）

## テキストコンテナ

### プロパティ

| プロパティ | 型 | 範囲 | 説明 |
|---|---|---|---|
| `xPosition` | number | 0–576 | 左端 |
| `yPosition` | number | 0–288 | 上端 |
| `width` | number | 0–576 | 幅 |
| `height` | number | 0–288 | 高さ |
| `borderWidth` | number | 0–5 | 枠線の太さ（0=なし） |
| `borderColor` | number | 0–15 | グレースケール（0=黒、15=白） |
| `borderRadius` | number | 0–10 | 角丸半径 |
| `paddingLength` | number | 0–32 | 内側余白（全辺均等） |
| `content` | string | — | 表示テキスト |

### テキストの制約

- 文字数制限: `createStartUpPageContainer` / `rebuildPageContainer` は**最大1000文字**、`textContainerUpgrade` は**最大2000文字**
- テキストはコンテナ幅で自動折り返し
- 改行は `\n` を使用
- 全画面テキストコンテナは約**400〜500文字**で埋まる
- テキスト配置: 常に左揃え（変更不可）
- フォントサイズ: 変更不可（固定）
- 太字・斜体: 不可

### ページ描画メソッドの使い分け

| メソッド | 用途 | フリッカー |
|---|---|---|
| `createStartUpPageContainer` | 起動時に一度だけ | — |
| `rebuildPageContainer` | レイアウト構造変更時（コンテナ追加/削除）| あり |
| `textContainerUpgrade` | テキスト内容の更新のみ | **なし** |

## リストコンテナ

ファームウェアが管理するスクロール可能なリスト。

```typescript
const list = new ListContainerProperty({
  xPosition: 0, yPosition: 0,
  width: 576, height: 288,
  containerID: 1, containerName: 'menu',
  isEventCapture: 1,
  itemContainer: new ListItemContainerProperty({
    itemCount: 3,
    itemWidth: 0,  // 0 = コンテナ幅に自動フィット
    isItemSelectBorderEn: 1,
    itemName: ['設定', 'ヘルプ', '終了'],
  }),
})
```

### リストの制約

- アイテム数: 1–20
- アイテム高さ: **40px固定**
- 上下スクロールはファームウェアが自動処理
- リスト内容の変更は **`rebuildPageContainer` でページごと再構築**が必要
- アイテムごとのスタイル変更不可

## 画像コンテナ

4ビットグレースケールのビットマップ画像を表示。

```typescript
// 1. コンテナを作成（この時点では空）
const img = new ImageContainerProperty({
  xPosition: 0, yPosition: 200,
  width: 100, height: 60,
  containerID: 2, containerName: 'icon',
})

// 2. ページ作成後、必ず updateImageRawData で内容を送る
await bridge.updateImageRawData({
  containerID: 2,
  containerName: 'icon',
  imageData: pixelData,  // number[] | Uint8Array | ArrayBuffer | base64文字列
})
```

### 画像の制約

- 幅: 20–288 px
- 高さ: 20–144 px
- 形式: 4ビットグレースケール（値0–15）
- **画像送信は直列** — 同時送信不可。必ず1つ完了後に次を送る
- **コンテナ作成後は必ず内容を送信** — 作成直後はプレースホルダー（空）

## GIFアニメーション（非公式・実験的）

SDK公式は「no animations」と明記していますが、以下の手順で疑似アニメーションが実現できます。

```typescript
import { parseGIF, decompressFrames } from 'gifuct-js'

async function playGif(bridge: EvenAppBridge, gifUrl: string) {
  const res = await fetch(gifUrl)
  const buf = await res.arrayBuffer()
  const frames = decompressFrames(parseGIF(buf), true)

  const canvas = document.createElement('canvas')
  // 30×30px程度まで縮小が必須（大きいと転送が追いつかない）
  canvas.width = 30
  canvas.height = 30
  const ctx = canvas.getContext('2d')!

  for (const frame of frames) {
    // フレームをCanvasに描画 → 4bitグレースケールに変換
    const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height)
    const pixelData = rgbaToGrayscale4bit(imageData.data)

    // 画像送信は直列（前の完了を待つ）
    await bridge.updateImageRawData({
      containerID: 1,
      containerName: 'gif',
      imageData: pixelData,
    })

    await new Promise(r => setTimeout(r, frame.delay))
  }
}
```

**制約:**
- **30×30px程度に縮小が必須** — BLE帯域の制約でそれ以上は転送が追いつかない
- FPSは低め（BLE転送時間に依存）
- 画像コンテナのサイズ制限（最大288×144px）は通常通り適用される

## よく使うUIパターン

### 疑似ボタン

`>` プレフィックスをカーソルインジケーターとして使用。

```
> 開始
  設定
  終了
```

### 選択ハイライト

テキストコンテナの `borderWidth` を切り替えて選択状態を示す（要 `rebuildPageContainer`）。

```typescript
// 選択中
{ borderWidth: 2, borderColor: 15 }
// 未選択
{ borderWidth: 0 }
```

### 3行レイアウト

288pxを3等分して縦に積み上げる。

```typescript
// 上段: y=0,   height=96
// 中段: y=96,  height=96
// 下段: y=192, height=96
```

### タイトルバー + コンテンツ

```typescript
const titleBar = { xPosition: 0, yPosition: 0,   width: 576, height: 40 }
const content  = { xPosition: 0, yPosition: 40,  width: 576, height: 248 }
```

### プログレスバー

Unicodeブロック文字を使って表現。

```typescript
const filled = Math.round((progress / 100) * barWidth)
const bar = '━'.repeat(filled) + '─'.repeat(barWidth - filled)
content = `進捗: ${bar} ${progress}%`
```

### ページめくり（長文）

約400〜500文字ごとに事前分割し、スクロール時に `rebuildPageContainer` で切り替える。

```typescript
const pages = paginateText(fullText, 450)
let currentPage = 0

async function showPage(index: number) {
  currentPage = index
  await bridge.rebuildPageContainer({
    containerTotalNum: 1,
    textObject: [{ ...containerConfig, content: pages[index] }],
  })
}
```

### 画像レイヤー + イベントキャプチャー

画像コンテナは `isEventCapture` 非対応。全画面テキストコンテナをイベント受付用に置く。

```typescript
// イベント受付用の透明テキストコンテナ（ユーザーには見えない）
const eventLayer = {
  xPosition: 0, yPosition: 0, width: 576, height: 288,
  containerID: 1, containerName: 'eventLayer',
  content: ' ',        // 空にできないので半角スペース
  isEventCapture: 1,
  borderWidth: 0, borderColor: 0, paddingLength: 0,
}

// 画像コンテナを重ねる
const imageLayer = {
  xPosition: 0, yPosition: 0, width: 200, height: 100,
  containerID: 2, containerName: 'display',
}
```

## 等幅レイアウト（ゲーム・グリッド向け）

G2のフォントはプロポーショナルのため、ASCII文字でグリッドを作ると列がずれます。**CJK全角文字**は全て同幅が保証されているため、ゲームや表形式のレイアウトに有効です。

```typescript
// ASCII → 全角変換（コードポイントに 0xFEE0 を加算）
function toFullwidth(str: string): string {
  return str.replace(/[\x20-\x7E]/g, (ch) =>
    ch === ' ' ? '　' : String.fromCharCode(ch.charCodeAt(0) + 0xFEE0)
  )
}
// 'A' → 'Ａ'、'0' → '０'、' ' → '　'（表意文字スペース U+3000）
```

グリッドの空セルには通常スペースではなく **`　`（表意文字スペース）** を使います。これにより空セルと文字セルの幅が一致します。

```typescript
// pong や snake などのゲームで使われるパターン
const EMPTY = '　'
const BALL  = '●'
const WALL  = '━'

function renderGrid(grid: string[][]): string {
  return grid.map(row => row.join('')).join('\n')
}
```

> **注意**: `▦`（U+25A6）や `●`（U+25CF）など一部の記号は全角文字と完全には幅が一致しない場合があります。ゲームグリッドでは許容範囲内です。

## 有用なUnicode文字

ファームウェアフォントに含まれる文字のみ表示されます。含まれない文字は**無音でスキップ**されます。

### 用途別早見表

| 用途 | 文字 |
|---|---|
| プログレスバー（横） | `━ ─` |
| プログレスバー（縦・8段階） | `█▇▆▅▄▃▂▁` |
| プログレスバー（左詰め・7段階） | `▉▊▋▌▍▎▏` |
| ナビゲーション | `▲△▶▷▼▽◀◁` |
| 選択インジケーター | `●○ ■□ ★☆` |
| 枠線（角丸） | `╭╮╯╰` |
| 枠線（直角） | `┌┐└┘ │─ ┼├┤┬┴` |
| カードスーツ | `♠♤ ♥♡ ♣♧` |
| 矢印 | `← → ↑ ↓ ↔ ↕ ⇒ ⇔` |
| その他 | `© ® ™ ∞ °` |

### 対応グリフ一覧

| 範囲 | 状況 |
|---|---|
| ASCII + Latin-1 (U+0020–U+00FF) | ほぼ完全（5文字除く: `¨ ¯ ´ µ ¸`） |
| 矢印 (U+2190–U+2199, U+21D2, U+21D4) | 12文字のみ |
| ボックス描画 (U+2500–U+2573) | 大部分対応（破線系は欠落） |
| ブロック要素 (U+2581–U+2595) | 下部・左部・フルブロックを中心に対応 |
| 幾何学図形 (U+25A0–U+25EF) | 一部対応（■□▲△▶▷▼▽●○◆◇★☆など） |
| 上付き・下付き数字 | 対応（⁰–⁹、₀–₉） |
| 分数 | `¼ ½ ⅛` のみ |
| 絵文字 (U+1F300+) | **非対応** |
| ダッシュ記号 (U+2700+) | **非対応** |

### 欠落している主な文字

```
▀ （上半ブロック）、▐ （右半ブロック）、░ ▓ （シェード）
▖▗▘▙▚▛▜▝▞▟ （四分割ブロック）
☀ ☁ ☂ ☃ （天気記号）
☠ ☢ ☣ ☮ ☯ ☺ ☻
```

## ベストプラクティス

- **頻繁な更新には `textContainerUpgrade`** を使う（カウンター・ステータス行・ライブデータ）
- **レイアウト変更時は `rebuildPageContainer`** — コンテナの追加・削除・種類変更・リスト内容変更時
- **`containerID` と `containerName` は必ず一致** させること — `textContainerUpgrade` でのミスマッチはサイレントに失敗
- **画像更新は1フレームあたり0.5〜2秒かかる** — BLE経由で無圧縮送信のため。アニメーションは設計段階から考慮
- **テキスト更新は画像更新より大幅に速い** — 即時性が必要な表示にはテキストを使う
- **全ブリッジ呼び出しを直列化** — `await` を必ず使う。並行呼び出しは接続クラッシュの原因になる

## フォント・テキスト計測

ピクセル精度のレイアウト計算には `@evenrealities/pretext` ライブラリを使用。詳細は [06-font-measurement.md](06-font-measurement.md) を参照。

```bash
npm install @evenrealities/pretext
```

```typescript
import { measureTextWrap } from '@evenrealities/pretext'

const result = measureTextWrap('こんにちは', 200)
// => { lineCount: 1, height: 27, lineWidths: [75] }
```

## フォンサイド（スマホ側）UIデザイン

グラスではなく、コンパニオンアプリ内の設定画面などを作る場合のカラートークン。

### カラートークン

| トークン | ライト | ダーク | 用途 |
|---|---|---|---|
| `--color-text` | `#232323` | `#FFFFFF` | プライマリテキスト |
| `--color-text-dim` | `#7B7B7B` | `#8A8A8A` | サブテキスト・キャプション |
| `--color-bg` | `#FFFFFF` | `#111111` | ページ背景 |
| `--color-surface` | `#EEEEEE` | `#1A1A1A` | カード・行の背景 |
| `--color-accent` | `#FEF991` | `#FEF991` | ブランドアクセント（ボタン・ハイライト） |

**注意:**
- `#FEF991`（Even ブランドイエロー）はアクセントのみ。ページ背景に使わない
- `#3CFA44`（グラス表示の緑）はG2ディスプレイ専用。スマホ側UIでは**絶対に使わない**

### タイポグラフィ

プライマリフォント: **FK Grotesk Neue**（CJKは Source Han Sans）

| スタイル | サイズ | ウェイト |
|---|---|---|
| Display | 34 px | 700 |
| Title | 24 px | 600 |
| Subtitle | 18 px | 500 |
| Body | 16 px | 400 |
| Caption | 13 px | 400 |
