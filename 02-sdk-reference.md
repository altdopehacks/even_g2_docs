# SDK リファレンス

`@evenrealities/even_hub_sdk` の全APIリファレンス（日本語版）。

## アーキテクチャ

```
Webアプリ (あなたのコード)
  ↕
EvenAppBridge (SDK)
  ↕
Even App (ホスト・コンパニオンアプリ)
  ↕
G2 グラス
```

アプリはVite + TypeScriptのWebページとして実装し、Flutter WebView内で動作します。

## インストール

```bash
npm install @evenrealities/even_hub_sdk
```

現在のバージョン: **0.0.10**

## 必須呼び出し順序

1. `waitForEvenAppBridge()` — ブリッジが準備完了するまで待機
2. `bridge.createStartUpPageContainer(container)` — **起動時に一度だけ**呼び出す
3. その後: `audioControl`、`imuControl`、イベントリスナー登録など

> `audioControl` と `imuControl` は `createStartUpPageContainer` が成功した後でないと動作しません。

## 初期化

### `waitForEvenAppBridge()` — 非同期（推奨）

ネイティブブリッジが準備完了したときに解決します。

```typescript
import { waitForEvenAppBridge } from '@evenrealities/even_hub_sdk'

const bridge = await waitForEvenAppBridge()
```

### `EvenAppBridge.getInstance()` — 同期シングルトン

`waitForEvenAppBridge()` 解決後のコールバック内など、ブリッジが既に初期化済みの場合のみ使用します。

```typescript
import { EvenAppBridge } from '@evenrealities/even_hub_sdk'

const bridge = EvenAppBridge.getInstance()
```

## メソッド一覧

| メソッド | 戻り値 | 説明 |
|---|---|---|
| `bridge.getUserInfo()` | `Promise<UserInfo>` | ログインユーザーのUID・名前・アバターURL・国を取得 |
| `bridge.getDeviceInfo()` | `Promise<DeviceInfo \| null>` | 接続中グラスのモデル・シリアル番号・ステータスを取得。未接続時は null |
| `bridge.setLocalStorage(key, value)` | `Promise<boolean>` | キーと値（文字列）のペアをストレージに保存 |
| `bridge.getLocalStorage(key)` | `Promise<string>` | キーで文字列を取得。未存在時は空文字列 |
| `bridge.createStartUpPageContainer(container)` | `Promise<StartUpPageCreateResult>` | 起動時に一度だけUIコンテナを定義。0=成功, 1=不正, 2=サイズ超過, 3=メモリ不足 |
| `bridge.rebuildPageContainer(container)` | `Promise<boolean>` | ページを完全に再描画（コンテナ追加・削除・切り替え時に使用） |
| `bridge.textContainerUpgrade(container)` | `Promise<boolean>` | テキストをちらつきなしでin-place更新（最大2000文字） |
| `bridge.updateImageRawData(data)` | `Promise<ImageRawDataUpdateResult>` | 画像コンテナにピクセルデータを送信。呼び出しは直列に行うこと |
| `bridge.shutDownPageContainer(exitMode?)` | `Promise<boolean>` | アプリを終了。0=即時終了, 1=終了確認ダイアログ表示 |
| `bridge.onLaunchSource(cb)` | `() => void` | 起動元イベントを購読。一度だけ発火: `'appMenu'` または `'glassesMenu'` |
| `bridge.onDeviceStatusChanged(cb)` | `() => void` | デバイスステータス変化を購読（接続・バッテリー・装着状態など） |
| `bridge.onEvenHubEvent(cb)` | `() => void` | 全ハブイベントを購読（listEvent, textEvent, sysEvent, audioEvent） |
| `bridge.audioControl(isOpen)` | `Promise<boolean>` | マイクを開く(`true`)または閉じる(`false`) |
| `bridge.imuControl(isOpen, reportFrq?)` | `Promise<boolean>` | IMUセンサーを有効/無効化。`reportFrq` は `ImuReportPace` 値 |
| `bridge.callEvenApp(method, params?)` | `Promise<any>` | ネイティブブリッジへの低レベル直接呼び出し |

## TypeScript インターフェース

### `TextContainerProperty`

テキスト表示コンテナ。

```typescript
class TextContainerProperty {
  xPosition?: number        // 0–576、左端からのオフセット
  yPosition?: number        // 0–288、上端からのオフセット
  width?: number            // 0–576、コンテナ幅
  height?: number           // 0–288、コンテナ高さ
  borderWidth?: number      // 0–5、枠線の太さ（px）
  borderColor?: number      // 0–15、グレースケールカラー
  borderRadius?: number     // 0–10、角丸半径
  paddingLength?: number    // 0–32、内側余白（px）
  containerID?: number      // ページ内で一意の整数ID
  containerName?: string    // 最大16文字
  isEventCapture?: number   // 0 or 1。ページ内でちょうど1つが1
  content?: string          // 初期テキスト（最大1000文字）
}
```

### `ListContainerProperty`

スクロール可能なリストコンテナ。

```typescript
class ListContainerProperty {
  xPosition?: number
  yPosition?: number
  width?: number
  height?: number
  borderWidth?: number
  borderColor?: number
  borderRadius?: number
  paddingLength?: number
  containerID?: number
  containerName?: string    // 最大16文字
  isEventCapture?: number
  itemContainer?: ListItemContainerProperty
}
```

### `ListItemContainerProperty`

リストコンテナ内のアイテム定義。

```typescript
class ListItemContainerProperty {
  itemCount?: number        // 1–20、アイテム数
  itemWidth?: number        // 0=自動（コンテナ幅に合わせる）
  isItemSelectBorderEn?: number  // 0 or 1。1=選択時に枠線表示
  itemName?: string[]       // アイテムラベル配列（最大20個、各64文字）
}
```

### `ImageContainerProperty`

画像表示コンテナ（作成後 `updateImageRawData` で内容を送信）。

```typescript
class ImageContainerProperty {
  xPosition?: number        // 0–576
  yPosition?: number        // 0–288
  width?: number            // 20–288
  height?: number           // 20–144
  containerID?: number
  containerName?: string    // 最大16文字
}
```

### `TextContainerUpgrade`

`bridge.textContainerUpgrade()` のパラメーター型。

```typescript
class TextContainerUpgrade {
  containerID?: number
  containerName?: string    // 最大16文字
  contentOffset?: number    // 書き込み開始文字位置
  contentLength?: number    // 置き換える文字数
  content?: string          // 新しいテキスト（最大2000文字）
}
```

### `CreateStartUpPageContainer`

`bridge.createStartUpPageContainer()` のパラメーター型。

```typescript
class CreateStartUpPageContainer {
  containerTotalNum?: number                    // 1–12、コンテナの合計数
  widgetId?: number                             // 自動割り当て（省略推奨）
  listObject?: ListContainerProperty[]          // リストコンテナ
  textObject?: TextContainerProperty[]          // テキストコンテナ（最大8個）
  imageObject?: ImageContainerProperty[]        // 画像コンテナ（最大4個）
}
```

### `UserInfo`

```typescript
interface UserInfo {
  uid: number      // 数値ユーザーID
  name: string     // 表示名
  avatar: string   // アバター画像URL
  country: string  // 国コード
}
```

### `DeviceInfo`

```typescript
interface DeviceInfo {
  readonly model: DeviceModel  // グラスのモデル識別子
  readonly sn: string          // シリアル番号
  status: DeviceStatus         // 現在の接続状態
}
```

### `DeviceStatus`

```typescript
interface DeviceStatus {
  sn: string
  connectType: DeviceConnectType
  isWearing?: boolean      // 装着中なら true
  batteryLevel?: number    // 0–100
  isCharging?: boolean     // 充電中なら true
  isInCase?: boolean       // ケース内なら true

  isNone(): boolean
  isConnected(): boolean
  isConnecting(): boolean
  isDisconnected(): boolean
  isConnectionFailed(): boolean
}
```

## イベントモデル

### `EvenHubEvent`

`bridge.onEvenHubEvent(cb)` のコールバック型。

```typescript
interface EvenHubEvent {
  listEvent?: List_ItemEvent
  textEvent?: Text_ItemEvent
  sysEvent?: Sys_ItemEvent
  audioEvent?: { audioPcm: Uint8Array }
  jsonData?: Record<string, any>
}
```

### `Text_ItemEvent`

テキストコンテナへのスクロール操作時に発火（クリックは `sysEvent` に来ることに注意）。

```typescript
interface Text_ItemEvent {
  containerID?: number
  containerName?: string
  eventType?: OsEventTypeList
}
```

### `List_ItemEvent`

リストアイテムの選択・スクロール時に発火。

```typescript
interface List_ItemEvent {
  containerID?: number
  containerName?: string
  currentSelectItemName?: string    // 選択中アイテムのラベル
  currentSelectItemIndex?: number   // 0始まりのインデックス
  eventType?: OsEventTypeList
}
```

### `Sys_ItemEvent`

IMUデータ・ライフサイクル信号などのシステムイベント。

```typescript
interface Sys_ItemEvent {
  eventType?: OsEventTypeList
  eventSource?: EventSourceType
  imuData?: IMU_Report_Data
  systemExitReasonCode?: number
}
```

## 列挙型（Enums）

```typescript
enum OsEventTypeList {
  CLICK_EVENT = 0,           // シングルプレス
  SCROLL_TOP_EVENT = 1,      // 上スワイプ
  SCROLL_BOTTOM_EVENT = 2,   // 下スワイプ
  DOUBLE_CLICK_EVENT = 3,    // ダブルプレス
  FOREGROUND_ENTER_EVENT = 4, // フォアグラウンドへ
  FOREGROUND_EXIT_EVENT = 5, // バックグラウンドへ
  ABNORMAL_EXIT_EVENT = 6,   // 異常終了
  SYSTEM_EXIT_EVENT = 7,     // システム終了（ユーザーが確認）
  IMU_DATA_REPORT = 8        // IMUデータ
}

enum ImuReportPace {
  P100 = 100,   // 100ms間隔
  P200 = 200,
  P300 = 300,
  P400 = 400,
  P500 = 500,   // 推奨（バランス良い更新頻度）
  P600 = 600,
  P700 = 700,
  P800 = 800,
  P900 = 900,
  P1000 = 1000  // 1000ms間隔
}

enum DeviceConnectType {
  None = 'none',
  Connecting = 'connecting',
  Connected = 'connected',
  Disconnected = 'disconnected',
  ConnectionFailed = 'connectionFailed'
}

enum EventSourceType {
  TOUCH_EVENT_FORM_DUMMY_NULL = 0,  // 不明
  TOUCH_EVENT_FROM_GLASSES_R = 1,   // G2 右アーム
  TOUCH_EVENT_FROM_RING = 2,        // R1 リング
  TOUCH_EVENT_FROM_GLASSES_L = 3    // G2 左アーム
}

enum DeviceModel {
  G1 = 'g1',
  G2 = 'g2',
  Ring1 = 'ring1'
}
```

## 戻り値コード

### `createStartUpPageContainer`

| コード | 名前 | 意味 |
|---|---|---|
| 0 | success | 成功 |
| 1 | invalid | パラメーター不正 |
| 2 | oversize | コンテンツが大きすぎる |
| 3 | outOfMemory | デバイスのメモリ不足 |

### `updateImageRawData`

| 値 | 意味 |
|---|---|
| `"success"` | 成功 |
| `"imageException"` | 汎用画像エラー |
| `"imageSizeInvalid"` | コンテナと画像サイズ不一致 |
| `"imageToGray4Failed"` | 4bitグレースケール変換失敗 |
| `"sendFailed"` | 送信エラー |

## 重要ルール

1. **`createStartUpPageContainer` は一度だけ** — 以降の再描画は `rebuildPageContainer` を使用
2. **`isEventCapture: 1` はページ内でちょうど1つ** — 0個または2個以上は動作不定
3. **コンテナ上限** — `containerTotalNum` は1–12、`textObject` 最大8個、`imageObject` 最大4個
4. **画像送信は直列** — `updateImageRawData` は並行呼び出し不可。必ず `await` してから次を呼ぶ
5. **画像コンテナは初期状態で空** — `createStartUpPageContainer` 後、必ず `updateImageRawData` で内容を送る
6. **`audioControl`・`imuControl` はstartup後** — `createStartUpPageContainer` 成功前に呼ぶと失敗
7. **イベントリスナーは必ず解除** — `onEvenHubEvent` などは解除関数を返す。コンポーネント破棄時に呼ぶこと
8. **`onLaunchSource` は一度だけ発火** — `waitForEvenAppBridge()` の直後に登録すること
