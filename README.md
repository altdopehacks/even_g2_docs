# Even Realities G2 開発ドキュメント

Even Realities G2スマートグラス向けアプリ開発のリファレンスドキュメント集です。

## Disclaimer

本ドキュメントはコミュニティメンバーが独自に調査・収集した情報をまとめたものです。

- Even Realities 社の公式ドキュメントではありません
- 内容の正確性・完全性・最新性を保証しません
- SDKやファームウェアのアップデートにより、記載内容が実際の動作と異なる場合があります
- 本ドキュメントを参照して生じたいかなる損害についても、作成者は責任を負いません
- BLEプロトコル関連の情報（08-ble-protocol.md）はリバースエンジニアリングに基づくものであり、特に変更される可能性があります

最新の公式情報は [Even Hub 公式ドキュメント](https://hub.evenrealities.com/docs/getting-started/overview) を参照してください。

## ドキュメント一覧

| ドキュメント | 内容 |
|---|---|
| [01-quickstart.md](01-quickstart.md) | プロジェクト作成からシミュレーター動作確認まで |
| [02-sdk-reference.md](02-sdk-reference.md) | SDK全メソッド・型・イベントのリファレンス |
| [03-ui-design.md](03-ui-design.md) | 576×288px表示向けUIデザインガイド |
| [04-input-handling.md](04-input-handling.md) | タッチパッド・リング入力とライフサイクルイベント |
| [05-device-features.md](05-device-features.md) | マイク・IMU・ストレージなどハードウェア機能 |
| [06-font-measurement.md](06-font-measurement.md) | ピクセル精度のテキストレイアウト計算 |
| [07-build-and-deploy.md](07-build-and-deploy.md) | パッケージングとEven Hubへの公開 |
| [08-ble-protocol.md](08-ble-protocol.md) | 低レベルBLEプロトコル（上級者向け・SDK非経由の直接制御） |
| [09-aivela-ring.md](09-aivela-ring.md) | AIVELA Ring Pro をG2入力デバイスとして使う方法・R1との比較 |

## G2 ハードウェア仕様

| 項目 | 仕様 |
|---|---|
| ディスプレイ解像度 | 576 × 288 px |
| カラー深度 | 4ビットグレースケール（16段階の緑） |
| カメラ | なし |
| スピーカー | なし |
| 接続方式 | Bluetooth 5.2 |
| 入力 | フレームのタッチパッド、オプションのR1リングコントローラー |

## Claude Code スキル一覧

このプロジェクトには `everything-evenhub` プラグインがインストールされています。

```bash
/quickstart <アプリ名>           # 新規プロジェクト作成
/template <アプリ名> --minimal   # テンプレートから作成
/glasses-ui "表示内容"           # UI実装
/handle-input "入力処理内容"     # 入力ハンドリング実装
/device-features "機能内容"      # ハードウェア機能実装
/font-measurement "計測内容"     # テキストサイズ計算
/test-with-simulator "内容"      # シミュレーターでテスト
/build-and-deploy                # ビルドとパッケージング
/sdk-reference <API名>           # SDK APIリファレンス参照
/design-guidelines "内容"        # デザインガイドライン参照
```

## Kotlin / Wasm での SDK 利用

JavaScript 以外の環境から Even Hub SDK を呼び出す場合、Kotlin Multiplatform + Compose を使った実装例があります（[EH-InNovel](https://github.com/even-realities/EH-InNovel)）。

アーキテクチャ:

```
Kotlin/Wasm コード → JS interop → @evenrealities/even_hub_sdk
```

Kotlin から JS ライブラリを呼び出す際は [`@JsModule` アノテーション](https://kotlinlang.org/docs/js-modules.html) でバインディングを定義します。SDK 自体は npm パッケージなので通常の `npm install` で導入可能です。

> ほとんどのアプリは TypeScript で十分です。Kotlin/Wasm は既存の Kotlin コードを流用したい場合や、Compose UI をそのまま使いたい場合に検討してください。

## スマホ側UIライブラリ（even-toolkit）

スマホのコンパニオンアプリUI（設定画面など）とグラス表示ロジックを両方カバーするデザインシステムです。

```bash
# 既存プロジェクトに追加
npm install even-toolkit

# 新規プロジェクトをスキャフォールディング（6テンプレート）
npx @even-toolkit/create-even-app my-app
```

### 主な機能

| 機能 | 内容 |
|---|---|
| Reactコンポーネント | 55個以上（Button / Card / ListItem / ChatContainer / DatePicker など） |
| ピクセルアートアイコン | 191個（6カテゴリ、32×32px） |
| `useGlasses()` フック | グラス表示ロジック・入力ハンドリング・終了処理を一括管理 |
| `useSTT()` フック | STT内蔵（Sonioxプロバイダー、G2マイク自動検出） |
| ナビゲーションヘルパー | `buildScrollableList()` / `moveHighlight()` / `wrapIndex()` |
| デザイントークン | 2025年ガイドライン準拠のカラー・タイポグラフィ |

### useGlasses アーキテクチャ

```typescript
import { useGlasses } from 'even-toolkit/useGlasses'

// 各スクリーンを display（表示）と action（入力処理）で定義
const homeScreen = {
  display(snapshot, nav) {
    return {
      lines: buildScrollableList({
        items: snapshot.items,
        highlightedIndex: nav.highlightedIndex,
        maxVisible: 5,
        formatter: (item) => item.title,
      }),
    }
  },
  action(action, nav, snapshot, ctx) {
    if (action === 'scrollDown') return { ...nav, highlightedIndex: nav.highlightedIndex + 1 }
  },
}
```

### useSTT フック（音声認識）

```typescript
import { useSTT } from 'even-toolkit'

const { transcript, isListening, start, stop } = useSTT({
  provider: 'soniox',
  language: 'ja',
  apiKey: 'your-key',
  vad: { silenceMs: 2500 },
})
// G2マイク・ブラウザマイク・カスタムソースを自動検出
```

### サンプルアプリ

| アプリ | 説明 |
|---|---|
| [EvenDemo](https://even-demo.vercel.app) | 全コンポーネントのショーケース |
| [EvenMarket](https://even-market.vercel.app) | リアルタイム株価 |
| [EvenKitchen](https://even-kitchen.vercel.app) | レシピ・調理ステップ |
| [EvenWorkout](https://even-workout.vercel.app) | トレーニングタイマー |
| [EvenBrowser](https://even-browser.vercel.app) | テキスト型Web閲覧 |

[GitHub](https://github.com/fabioglimb/even-toolkit)

## 参考アプリ（OSSサンプル）

| アプリ | 特徴 | リポジトリ |
|---|---|---|
| demo | 全コンテナ・全イベントのショーケース。**まず読むべき** | [nickustinov/demo-app-g2](https://github.com/nickustinov/demo-app-g2) |
| chess | テスト・lint・モジュール設計の参考 | [dmyster145/EvenChess](https://github.com/dmyster145/EvenChess) |
| reddit | APIプロキシ・パッケージングの参考 | [fuutott/rdt-even-g2-rddit-client](https://github.com/fuutott/rdt-even-g2-rddit-client) |
| weather | even-toolkit を使った設定UIの参考 | [nickustinov/weather-even-g2](https://github.com/nickustinov/weather-even-g2) |
| tesla | バックエンドサーバー・画像レンダリング | [nickustinov/tesla-even-g2](https://github.com/nickustinov/tesla-even-g2) |
| pong / snake | キャンバスレンダリングのゲーム | [pong](https://github.com/nickustinov/pong-even-g2) / [snake](https://github.com/nickustinov/snake-even-g2) |
| flappy-g2 | タップ操作ゲーム・4bitグレースケール | [200even/flappy-g2](https://github.com/200even/flappy-g2) |

## 関連リンク

- [Even Hub 公式ドキュメント](https://hub.evenrealities.com/docs/getting-started/overview)
- [even_hub_sdk (npm)](https://www.npmjs.com/package/@evenrealities/even_hub_sdk)
- [evenhub-simulator (npm)](https://www.npmjs.com/package/@evenrealities/evenhub-simulator)
- [evenhub-cli (npm)](https://www.npmjs.com/package/@evenrealities/evenhub-cli)
- [コミュニティ Discord](https://discord.gg/Y4jHMCU4sv)
- [even-g2-notes (コミュニティ資料)](https://github.com/nickustinov/even-g2-notes)
- [even-dev (開発環境)](https://github.com/BxNxM/even-dev) — 複数アプリを切り替えられる統合ランチャー。21本のコミュニティアプリカタログ（`apps.json`）付き。`AUDIO_DEVICE` 変数でシミュレーターへのマイク入力テストも可能
