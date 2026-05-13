# フォント計測

`@evenrealities/pretext` を使ったピクセル精度のテキストレイアウト計算。G2ファームウェアのLVGLレンダリングエンジンと一致する結果を得られます。

## インストール

```bash
npm install @evenrealities/pretext
```

## ディスプレイ定数

| 定数 | 値 |
|---|---|
| 画面サイズ | 576 × 288 px |
| 行高 | **27 px**（固定） |
| リストアイテム高 | **40 px**（固定） |
| リストアイテム水平パディング | 12 px（片側） |

## API

### `getTextWidth(text: string): number`

単一行テキストのピクセル幅を返します（カーニング適用、折り返しなし）。

```typescript
import { getTextWidth } from '@evenrealities/pretext'

const width = getTextWidth('Hello, world!')  // => 79
const jpWidth = getTextWidth('こんにちは')  // => XXpx
```

### `measureTextWrap(text: string, maxWidth: number): MeasureTextResult`

折り返しを含むテキストのレイアウトを計測します。`maxWidth` はパディング・枠線を引いた**内側の幅**を指定してください。

```typescript
import { measureTextWrap } from '@evenrealities/pretext'

const result = measureTextWrap('The quick brown fox jumps over the lazy dog', 200)
// => { lineCount: 2, height: 54, lineWidths: [192, 96] }
```

**戻り値:**

| フィールド | 型 | 説明 |
|---|---|---|
| `lineCount` | number | 折り返し後の行数 |
| `height` | number | テキスト総高さ（`lineCount * 27`） |
| `lineWidths` | number[] | 各行のピクセル幅 |

### `pxTruncate(text: string, maxPx: number): string`

指定ピクセル幅に収まるようテキストを切り詰め、必要なら `'...'` を追加します。

```typescript
import { pxTruncate } from '@evenrealities/pretext'

const label = pxTruncate('Hello, world!', 50)  // => 'Hell...'
const fits  = pxTruncate('Hi', 50)             // => 'Hi'
```

> この関数は単一行専用です。複数行のテキストは各行を個別に処理してください。

## パディングと枠線の影響

コンテナに `paddingLength` や `borderWidth` がある場合、LVGLレンダラーはテキストエリアを縮小します。コンテナ幅をそのまま `maxWidth` に使うと内容がはみ出します。

```
コンテナ (width × height)
┌─ 枠線 (borderWidth px) ─────────────────────┐
│ ┌─ パディング (paddingLength px) ──────────┐ │
│ │                                          │ │
│ │  テキスト描画エリア                        │ │
│ │  innerWidth  = width  - 2*(pad + border) │ │
│ │  innerHeight = height - 2*(pad + border) │ │
│ │                                          │ │
│ └──────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

### 計算式

```typescript
const inset = paddingLength + borderWidth
const innerW = containerWidth - 2 * inset
const innerH = containerHeight - 2 * inset
```

### よくある間違い

```typescript
// ❌ 間違い — コンテナ幅をそのまま使用
const m = measureTextWrap(text, 560)

// ✅ 正しい — パディングと枠線を引く
const m = measureTextWrap(text, 560 - 2 * (padding + border))
```

## よく使うパターン

### テキストに合わせてコンテナサイズを決める

```typescript
import { measureTextWrap } from '@evenrealities/pretext'

const containerWidth = 300
const padding = 8
const innerW = containerWidth - 2 * padding

const result = measureTextWrap(myText, innerW)
const containerHeight = result.height + 2 * padding

// コンテナ定義に使用
const textContainer = {
  xPosition: 0, yPosition: 0,
  width: containerWidth,
  height: containerHeight,
  paddingLength: padding,
  // ...
}
```

### 全画面コンテナに何行入るか計算する

```typescript
import { measureTextWrap } from '@evenrealities/pretext'

const containerW = 576
const containerH = 288
const padding = 4

const innerW = containerW - 2 * padding
const innerH = containerH - 2 * padding
const maxLines = Math.floor(innerH / 27)  // 行高は27px固定

// maxLines行に収まるようにテキストを切る
const m = measureTextWrap(text, innerW)
console.log(`${m.lineCount}行 / 最大${maxLines}行`)
```

### テキストを行数でページング

```typescript
import { measureTextWrap } from '@evenrealities/pretext'

function paginateText(text: string, innerW: number, maxLines: number): string[] {
  const words = text.split(' ')
  const pages: string[] = []
  let current = ''

  for (const word of words) {
    const candidate = current ? `${current} ${word}` : word
    const { lineCount } = measureTextWrap(candidate, innerW)
    if (lineCount > maxLines && current) {
      pages.push(current)
      current = word
    } else {
      current = candidate
    }
  }
  if (current) pages.push(current)
  return pages
}
```

### リストコンテナの高さを計算する

```typescript
const itemCount = 5
const padding = 4                                // paddingLength
const listHeight = itemCount * 40 + 2 * padding  // アイテム高40px + パディング
```

### テキストを1行に収まるよう切り詰める

```typescript
import { pxTruncate } from '@evenrealities/pretext'

const containerWidth = 300
const padding = 4
const innerW = containerWidth - 2 * padding

const label = pxTruncate(longText, innerW)
```

## 実践例: ヘッダー + コンテンツのレイアウト

```typescript
import { measureTextWrap, pxTruncate } from '@evenrealities/pretext'

const SCREEN_W = 576
const SCREEN_H = 288
const PADDING = 4

// ヘッダー（1行固定）
const headerH = 27 + 2 * PADDING  // 35px
const headerInnerW = SCREEN_W - 2 * PADDING

// コンテンツエリア
const contentW = SCREEN_W
const contentH = SCREEN_H - headerH
const contentInnerW = contentW - 2 * PADDING
const contentInnerH = contentH - 2 * PADDING
const maxLines = Math.floor(contentInnerH / 27)

// ヘッダーテキストを1行に切り詰め
const headerText = pxTruncate(title, headerInnerW)

// コンテンツテキストの計測
const m = measureTextWrap(body, contentInnerW)
const actualLines = Math.min(m.lineCount, maxLines)
```
