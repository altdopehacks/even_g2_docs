# AIVELA Ring Pro と G2 の連携

サードパーティ製スマートリング「AIVELA Ring Pro」をEven G2アプリの入力デバイスとして使う方法。

> **注意**: AIVELA Ring Pro は Even Realities 社の製品ではありません。本ドキュメントは実機調査とコミュニティ情報に基づきます。ファームウェアアップデートにより動作が変わる可能性があります。

## デバイス概要

| 項目 | 仕様 |
|---|---|
| 接続方式 | Bluetooth 5.0 |
| センサー | 光学式フィンガーナビゲーションセンサー（PixArt） |
| チップ | Ambiq Apollo 4 SoC |
| 対応OS | iOS 15.0以上 / Android 9.0以上 |
| バッテリー | 約7日間連続使用 |
| 公式アプリ | AIVELA（iOS/Android） |

## Bluetooth HID として動作する

AIVELA Ring Pro は Bluetooth HID（Human Interface Device）プロファイルで接続されます。iOS の Bluetooth 設定で「Input device: ON」と表示されることで確認済みです。

```
AIVELA Ring Pro
  → Bluetooth HID → iPhone（OSドライバーレベル）
                       ↓
                 WKWebView → keydown / keyup イベント
```

HID はOS最下層のドライバーで処理されるため：
- AIVELA 専用アプリが起動していなくても動作する
- Even Hub の WKWebView に直接キーイベントが届く
- PC（Mac/Windows）にも同様に接続できる

## 8つのタッチジェスチャー

| ジェスチャー | 動作 |
|---|---|
| Single Tap | 1回タップ |
| Double Tap | 2回タップ |
| Triple Tap | 3回タップ |
| Long Press | 長押し（一定時間後に1発のイベント） |
| Swipe Up | 上スワイプ |
| Swipe Down | 下スワイプ |
| Swipe Left | 左スワイプ |
| Swipe Right | 右スワイプ |

**Long Press は「押し続けている間ずっと発火」ではなく、一定時間後に1回だけ発火するイベントです。** push-to-talk（ホールド中に録音）には使えません。

### タッチパッドのスリープ

バッテリー節約のため、1分間操作がないとタッチパッドがスリープします。復帰させるには上下を素早くスワイプします。G2アプリ使用前に習慣として覚えておく必要があります。

## タッチモード

AIVELAアプリで切り替えられる3つのモードがあります。モードによって送信されるHIDキーコードが変わります。

| モード | ジェスチャー → HIDキー | WebViewに届くか |
|---|---|---|
| **Music / Photo** | Swipe Up/Down → Volume Up/Down | ❌ iOSがシステム音量として消費 |
| **Short Video** | Swipe Up/Down → Volume Up/Down | ❌ 同上 |
| **Slide / E-Book** | Swipe Up/Left → ArrowUp/Left | ✅ |
| **Slide / E-Book** | Swipe Down/Right → ArrowDown/Right | ✅ |

### なぜ Volume キーは届かないか

Volume Up/Down キーは iOS のシステムが最優先で処理（consume）します。Spotify などの音量が変わるのはシステム音量が変わるためであり、アプリにキーイベントが届いているわけではありません。WKWebView には `keydown` イベントとして伝播しません。

```
Volume Up HID キー
  → iOS システム音量変更（Spotifyの音が大きくなる）✅
  → WKWebView keydown イベント ❌（届かない）

ArrowRight HID キー
  → iOS がスルー
  → WKWebView keydown { code: 'ArrowRight' } ✅（届く）
```

## G2アプリでの設定方法

1. AIVELAアプリ → Touch Settings → **Slide / E-Book** を選択
2. Even Hub アプリを開いてG2に接続
3. WebView内で `keydown` イベントをリッスン

```typescript
document.addEventListener('keydown', (e) => {
  e.preventDefault()
  console.log(e.code)  // 実機で各ジェスチャーのキーコードを確認
})
```

## 推奨ジェスチャーマッピング（G2アプリ用）

| ジェスチャー | 推奨割り当て |
|---|---|
| Swipe Up | G2リスト上スクロール / 前のページ |
| Swipe Down | G2リスト下スクロール / 次のページ |
| Swipe Left | 前の画面に戻る |
| Swipe Right | 次の画面へ / 選択確定 |
| Single Tap | 選択 / 決定 |
| Double Tap | ダブルタップ（`shutDownPageContainer(1)` 用途など） |
| **Long Press** | **音声コマンド起動（VAD自動停止と組み合わせる）** |
| Triple Tap | 予備（将来用） |

## 音声コマンドの実装例

Long Press で録音開始、VAD（音声検出）で自動停止するパターン。

```typescript
import { MicVAD } from '@ricky0123/vad-web'

const vad = await MicVAD.new({
  onSpeechEnd: async (audio: Float32Array) => {
    vad.pause()
    await bridge.audioControl(false)
    await bridge.textContainerUpgrade({ containerID: 1, content: '⏳ 処理中...' })
    const text = await transcribe(audio)   // Deepgram / Whisper 等
    const reply = await askLlm(text)       // Groq 等
    await bridge.textContainerUpgrade({ containerID: 1, content: reply })
  },
})

document.addEventListener('keydown', async (e) => {
  e.preventDefault()
  switch (e.code) {
    case 'ArrowUp':
      scrollUp()
      break
    case 'ArrowDown':
      scrollDown()
      break
    case '???':  // Long Press のキーコード（実機確認後に差し替え）
      await bridge.audioControl(true)
      await bridge.textContainerUpgrade({ containerID: 1, content: '● 話してください...' })
      vad.start()
      break
  }
})
```

## R1（Even Realities 純正リング）との比較

| 項目 | R1 | AIVELA Ring Pro |
|---|---|---|
| G2との接続 | R1 → G2 → iPhone（2ホップ） | AIVELA → iPhone（1ホップ） |
| 入力レイテンシ | やや高い（BLE 2段） | 低い（HIDドライバー直結） |
| G2状態の影響 | ディスプレイ更新中に遅延の可能性 | 影響なし |
| ジェスチャー数 | 4種類 | **8種類** |
| 左右スワイプ | ❌ | ✅ |
| Long Press | ❌ | ✅ |
| イベント取得方法 | `onEvenHubEvent`（SDK） | `keydown`（WebView） |
| ListContainer自動スクロール | ✅ ファームウェア自動処理 | ❌ JS で手動実装が必要 |
| 音楽・スライド操作 | ❌ G2専用 | ✅ 汎用リモコンとして使える |
| PC接続 | ❌ | ✅ |

### ListContainer の制約について

R1 の上下スワイプは Even Hub SDK 経由で G2 ファームウェアに届き、ListContainer を自動スクロールします。AIVELA のキーイベントはファームウェアに届かないため、ListContainer の自動スクロールは動作しません。

回避策: ListContainer を使わず TextContainer + `textContainerUpgrade` で手動スクロールを実装します。

```typescript
let scrollOffset = 0
const LINES_PER_PAGE = 7

document.addEventListener('keydown', (e) => {
  if (e.code === 'ArrowDown') {
    scrollOffset++
    renderContent()
  } else if (e.code === 'ArrowUp') {
    scrollOffset = Math.max(0, scrollOffset - 1)
    renderContent()
  }
})

async function renderContent() {
  const lines = allLines.slice(scrollOffset, scrollOffset + LINES_PER_PAGE)
  await bridge.textContainerUpgrade({
    containerID: 1,
    content: lines.join('\n'),
  })
}
```

## 参考リンク

- [AIVELA 公式サイト](https://www.aivela.com)
- [AIVELA Ring Pro ユーザーガイド](https://www.aivela.com/pages/aivela-ring-pro-user-guide)
- [Makuake クラウドファンディングページ](https://www.makuake.com/project/aivela_ring_pro/)
