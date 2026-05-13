# 入力ハンドリング

G2グラスのタッチパッドジェスチャー・R1リング入力・ライフサイクルイベントの実装ガイド。

## 入力ソース

| ソース | ジェスチャー | 備考 |
|---|---|---|
| G2 タッチパッド（テンプル） | プレス、ダブルプレス、上スワイプ、下スワイプ | 主入力 |
| R1 タッチパッド（リング） | プレス、ダブルプレス、上スワイプ、下スワイプ | オプション周辺機器。G2と同じジェスチャーセット |
| IMU（加速度計/ジャイロ） | 頭部の向き、モーションデータ | [05-device-features.md](05-device-features.md) 参照 |

## イベントルーティング

イベントは**アクティブなコンテナの種類**（`isEventCapture: 1` を持つもの）によって異なるフィールドに届きます。

### テキストコンテナがアクティブの場合

| ジェスチャー | イベントフィールド | eventType |
|---|---|---|
| 上スワイプ | `event.textEvent` | `1` (SCROLL_TOP_EVENT) |
| 下スワイプ | `event.textEvent` | `2` (SCROLL_BOTTOM_EVENT) |
| シングルプレス | `event.sysEvent` | `undefined` / `0` |
| ダブルプレス | `event.sysEvent` | `3` (DOUBLE_CLICK_EVENT) |

> **重要**: テキストコンテナへのクリック・ダブルクリックは `textEvent` ではなく **`sysEvent`** に届きます。`textEvent` に来るのはスクロールジェスチャーのみです。これは最も多い実装バグの原因です。

### リストコンテナがアクティブの場合

| ジェスチャー | イベントフィールド | 詳細 |
|---|---|---|
| 上/下スワイプ | （内部処理） | SDKがリストを自動スクロール。イベントは発火しない |
| シングルプレス | `event.listEvent` | `.currentSelectItemIndex` = 選択アイテムのインデックス |
| ダブルプレス | `event.sysEvent` | `.eventType` = `3` |

### システムイベント（常に利用可能）

| 種類 | イベントフィールド | eventType |
|---|---|---|
| フォアグラウンド開始 | `event.sysEvent` | `4` |
| フォアグラウンド終了 | `event.sysEvent` | `5` |
| 異常終了 | `event.sysEvent` | `6` |
| システム終了 | `event.sysEvent` | `7` |

## OsEventTypeList 一覧

| 値 | 名前 | 説明 |
|---|---|---|
| 0 | CLICK_EVENT | シングルプレス（G2またはR1） |
| 1 | SCROLL_TOP_EVENT | 上スワイプ |
| 2 | SCROLL_BOTTOM_EVENT | 下スワイプ |
| 3 | DOUBLE_CLICK_EVENT | ダブルプレス（G2またはR1） |
| 4 | FOREGROUND_ENTER_EVENT | アプリがフォアグラウンドに |
| 5 | FOREGROUND_EXIT_EVENT | アプリがバックグラウンドに |
| 6 | ABNORMAL_EXIT_EVENT | 予期しない切断 |
| 7 | SYSTEM_EXIT_EVENT | システム終了（ユーザーが終了ダイアログを確認） |
| 8 | IMU_DATA_REPORT | IMUデータサンプル |

## Protobuf ゼロ値の省略に注意

SDKは内部でprotobufを使用しています。**ゼロ/デフォルト値（0, false, 空文字列）のフィールドは `undefined` になります**。

影響を受けるフィールド:
- `sysEvent.eventType` — シングルクリックは `0` だが、`undefined` として届く
- `listEvent.currentSelectItemIndex` — 最初のアイテムは `0` だが、`undefined` として届く

**必ず Null合体演算子を使用すること:**

```typescript
const type = event.sysEvent?.eventType ?? 0          // undefinedを0として扱う
const idx = event.listEvent?.currentSelectItemIndex ?? 0  // undefinedを0として扱う
```

## 完全なイベントハンドリングテンプレート

```typescript
import { waitForEvenAppBridge } from '@evenrealities/even_hub_sdk'

const bridge = await waitForEvenAppBridge()

const unsubscribe = bridge.onEvenHubEvent(event => {
  // リストアイテム選択（リストコンテナへのシングルプレス）
  if (event.listEvent) {
    const idx = event.listEvent.currentSelectItemIndex ?? 0
    const name = event.listEvent.currentSelectItemName ?? ''
    console.log(`選択: [${idx}] ${name}`)
    return
  }

  // テキストコンテナへのスクロール（クリックはここには来ない）
  if (event.textEvent) {
    const type = event.textEvent.eventType ?? 0
    if (type === 1) {
      console.log('上スワイプ')
    } else if (type === 2) {
      console.log('下スワイプ')
    }
    return
  }

  // システムイベント（クリック、ダブルクリック、ライフサイクル）
  if (event.sysEvent) {
    const type = event.sysEvent.eventType ?? 0
    switch (type) {
      case 0: // CLICK_EVENT
        console.log('シングルプレス')
        break
      case 3: // DOUBLE_CLICK_EVENT
        // ダブルプレスで終了確認ダイアログを表示
        bridge.shutDownPageContainer(1)
        break
      case 4: // FOREGROUND_ENTER_EVENT
        // 再フォアグラウンド — タイマーや表示を再開
        resumeApp()
        break
      case 5: // FOREGROUND_EXIT_EVENT
        // バックグラウンドへ — 状態を保存
        saveState()
        break
      case 6: // ABNORMAL_EXIT_EVENT
      case 7: // SYSTEM_EXIT_EVENT
        // クリーンアップ
        cleanup()
        break
    }
    return
  }
})

// コンポーネント破棄時に必ず解除
// unsubscribe()
```

## G2 vs R1 の区別

G2（テンプルタッチパッド）とR1（リングタッチパッド）は同じジェスチャーセットを持ちます。どちらからの入力かを区別するには `Sys_ItemEvent` の `eventSource` を確認します。

```typescript
import { EventSourceType } from '@evenrealities/even_hub_sdk'

if (event.sysEvent) {
  switch (event.sysEvent.eventSource) {
    case EventSourceType.TOUCH_EVENT_FROM_GLASSES_R:  // 1: G2 右アーム
    case EventSourceType.TOUCH_EVENT_FROM_GLASSES_L:  // 3: G2 左アーム
      // グラスからの入力
      break
    case EventSourceType.TOUCH_EVENT_FROM_RING:       // 2: R1 リング
      // リングからの入力
      break
  }
}
```

## 終了処理

全てのアプリはグラス/リング操作で終了できる手段を提供すること。自前の確認UIを作らず、SDKのシステム終了ダイアログを使用することを推奨します。

### 推奨パターン: ダブルタップでシステム終了ダイアログ

```typescript
if (eventType === 3) { // DOUBLE_CLICK_EVENT
  // システム終了ダイアログを表示
  // ここではリソースを解放しない — ユーザーがキャンセルできるため
  // 確認された場合は SYSTEM_EXIT_EVENT (7) が発火する
  bridge.shutDownPageContainer(1)
  return
}
```

> **注意**: `shutDownPageContainer(1)` を呼ぶ前にリスナー解除やハードウェア停止をしないこと。ユーザーがキャンセルした場合、アプリが画面に残ったまま入力を受け付けなくなります。クリーンアップは `ABNORMAL_EXIT_EVENT` / `SYSTEM_EXIT_EVENT` ハンドラー内で行います。

## ストア審査リジェクト条件（重要）

Even Hub の審査チームは、**ルートページでダブルタップしたときに `shutDownPageContainer(1)` を呼ばないアプリをリジェクトします。**

審査コメント:
> "Please ensure double tapping at the root page on OS can invoke exit dialogue (shutDownContainer(1))."

### 必須実装パターン

```typescript
function onEvent(event: EvenHubEvent) {
  const type = event.sysEvent?.eventType ?? 0

  if (type === OsEventTypeList.DOUBLE_CLICK_EVENT) {
    if (state.screen === 'root') {
      // ルートページ: exitMode=1 でシステム終了ダイアログを表示
      // exitMode=0（即時終了）はNG — 審査リジェクト対象
      void bridge.shutDownPageContainer(1)
      return
    }
    // 非ルートページ: 1つ前の画面に戻る
    void goBack()
    return
  }
}
```

### チェックリスト

- [ ] ルートページでダブルタップ → `shutDownPageContainer(1)` を呼ぶ
- [ ] `exitMode` は `1`（`0` は不可）
- [ ] 非ルートページでダブルタップ → 前の画面に戻る
- [ ] `even-toolkit` の `useGlasses()` フックを使う場合は自動で対応済み

## ライフサイクルイベントの使い方

| イベント | いつ使うか |
|---|---|
| `FOREGROUND_ENTER_EVENT` (4) | 現在の状態を再描画。タイマー・IMUを再開 |
| `FOREGROUND_EXIT_EVENT` (5) | 保留中の状態を `setLocalStorage` に保存。タイマーを停止 |
| `ABNORMAL_EXIT_EVENT` (6) | ハードウェア停止（`imuControl(false)`, `audioControl(false)`）、リスナー解除、状態保存 |
| `SYSTEM_EXIT_EVENT` (7) | `ABNORMAL_EXIT_EVENT` と同じクリーンアップ処理 |
