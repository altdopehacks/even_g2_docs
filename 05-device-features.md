# デバイス機能

G2のハードウェア機能（マイク・IMU・デバイス情報・ユーザー情報・ローカルストレージ）の実装ガイド。

## 前提条件

`createStartUpPageContainer` が成功した後でないと `audioControl` と `imuControl` は動作しません。

## マイク（音声キャプチャー）

`bridge.audioControl(true)` でマイクを開き、`bridge.audioControl(false)` で閉じます。

音声データは `onEvenHubEvent` を通じて届きます。`event.audioEvent.audioPcm` は `Uint8Array` です。

**フォーマット:** PCM S16LE（符号付き16ビットリトルエンディアン）、モノラル、16kHz、**フレーム長10ms・1フレーム40バイト**

### 内部コーデック（LC3）

BLE回線上では音声データは **LC3（Bluetooth LE Audio）コーデック**で転送されています。しかし Even Hub SDK がデコード処理を内部で行うため、`audioPcm` には **デコード済みのPCMデータ**が届きます。アプリ側でLC3デコードを実装する必要はありません。

`getUserMedia` でG2マイクにアクセスすることはできません。音声データは `onEvenHubEvent` 経由でのみ取得可能です。

```typescript
// 前提: createStartUpPageContainer が成功していること
await bridge.audioControl(true)

const unsubscribe = bridge.onEvenHubEvent(event => {
  if (event.audioEvent) {
    const pcm = event.audioEvent.audioPcm  // Uint8Array
    // PCMデータを処理（Web Audio API、音声認識など）
    processPCM(pcm)
  }
})

// 停止とクリーンアップ
await bridge.audioControl(false)
unsubscribe()
```

### app.json での権限設定

マイク使用にはアプリマニフェストに権限を追加する必要があります。

```json
"permissions": [
  {
    "name": "g2-microphone",
    "desc": "音声コマンドのためのマイク使用"
  }
]
```

G2グラスのマイクには `g2-microphone`、ペアリング中のスマートフォンのマイクには `phone-microphone` を使います。

## IMU（モーションセンサー）

`bridge.imuControl(true, ImuReportPace.P500)` でIMUレポートを開始、`bridge.imuControl(false)` で停止します。

IMUデータは `onEvenHubEvent` 経由で届きます。`event.sysEvent.imuData` に `{ x, y, z }` の値が入ります。

```typescript
import { ImuReportPace, OsEventTypeList } from '@evenrealities/even_hub_sdk'

// 前提: createStartUpPageContainer が成功していること
await bridge.imuControl(true, ImuReportPace.P500)

const unsubscribe = bridge.onEvenHubEvent(event => {
  const sys = event.sysEvent
  if (!sys?.imuData) return
  if (sys.eventType !== OsEventTypeList.IMU_DATA_REPORT) return

  const { x, y, z } = sys.imuData
  console.log(`IMU: x=${x}, y=${y}, z=${z}`)
})

// 停止
await bridge.imuControl(false)
unsubscribe()
```

### ImuReportPace 値

| 値 | 説明 |
|---|---|
| `ImuReportPace.P100` | 100ms間隔（高頻度） |
| `ImuReportPace.P200` | 200ms間隔 |
| `ImuReportPace.P300` | 300ms間隔 |
| `ImuReportPace.P400` | 400ms間隔 |
| `ImuReportPace.P500` | 500ms間隔（推奨・バランス） |
| `ImuReportPace.P1000` | 1000ms間隔（低頻度） |

## デバイス情報

```typescript
const deviceInfo = await bridge.getDeviceInfo()

if (deviceInfo) {
  console.log('モデル:', deviceInfo.model)        // 'g2'
  console.log('シリアル番号:', deviceInfo.sn)
  console.log('接続状態:', deviceInfo.status.isConnected())
  console.log('バッテリー:', deviceInfo.status.batteryLevel, '%')
  console.log('装着中:', deviceInfo.status.isWearing)
}
```

### DeviceStatus のヘルパーメソッド

| メソッド | 戻り値 | 説明 |
|---|---|---|
| `isNone()` | boolean | 状態なし |
| `isConnected()` | boolean | 接続済み |
| `isConnecting()` | boolean | 接続中 |
| `isDisconnected()` | boolean | 切断済み |
| `isConnectionFailed()` | boolean | 接続失敗 |

### リアルタイムステータス監視

```typescript
const unsubscribe = bridge.onDeviceStatusChanged(status => {
  console.log('バッテリー:', status.batteryLevel, '%')
  console.log('接続:', status.isConnected())
  console.log('装着:', status.isWearing)
})

// 不要になったら解除
// unsubscribe()
```

## ユーザー情報

```typescript
const userInfo = await bridge.getUserInfo()

console.log('UID:', userInfo.uid)
console.log('名前:', userInfo.name)
console.log('アバター:', userInfo.avatar)   // URL
console.log('国:', userInfo.country)
```

## ローカルストレージ

Even App に永続化するキー・バリューストア（アプリ再起動後も保持）。

```typescript
// 保存
const success = await bridge.setLocalStorage('settings.volume', '80')

// 読み取り
const volume = await bridge.getLocalStorage('settings.volume')
// キーが存在しない場合は空文字列 '' が返る
```

### 重要: ブラウザ標準のストレージは使わない

Even AppはFlutter WebView（`flutter_inappwebview`）上で動作します。**ブラウザの `localStorage` や `IndexedDB` はアプリ再起動後に消えることがあります**。

ユーザーの設定・進捗・お気に入り・キャッシュなど全ての永続データには `bridge.setLocalStorage` / `bridge.getLocalStorage` を使用してください。

### キーの削除

`removeLocalStorage` は存在しません。空文字列を書き込んで「存在しない」として扱うのが慣例です。

```typescript
await bridge.setLocalStorage('my.key', '')  // 削除の代わり
const value = await bridge.getLocalStorage('my.key')
if (!value) { /* 存在しないとして扱う */ }
```

### 推奨パターン: 起動時キャッシュ

ブリッジ呼び出しは非同期のため、UIレンダリング中に逐次読み込むと遅延が発生します。起動時に全キーをメモリにロードし、同期的に読み書きするパターンが推奨されています。

```typescript
const cache = new Map<string, string>()

// 起動時（bridge接続後・UI描画前）に一括ロード
async function initStorage(bridge: EvenAppBridge, keys: string[]) {
  await Promise.all(keys.map(async (key) => {
    const value = await bridge.getLocalStorage(key)
    if (value) cache.set(key, value)
  }))
}

// 同期読み取り（UIから呼べる）
function getItem(key: string): string | null {
  return cache.get(key) ?? null
}

// ライトスルー: キャッシュをすぐ更新し、永続化はバックグラウンドで
function setItem(bridge: EvenAppBridge, key: string, value: string) {
  cache.set(key, value)
  void bridge.setLocalStorage(key, value)
}
```

### 大きなデータのチャンク保存

```typescript
const CHUNK_SIZE = 50_000  // 1キーあたりの文字数
const PREFIX = 'myapp.content_'

async function saveContent(bridge: EvenAppBridge, id: string, text: string) {
  const chunks = Math.ceil(text.length / CHUNK_SIZE)
  await bridge.setLocalStorage(`${PREFIX}${id}_n`, String(chunks))

  for (let i = 0; i < chunks; i++) {
    await bridge.setLocalStorage(
      `${PREFIX}${id}_${i}`,
      text.slice(i * CHUNK_SIZE, (i + 1) * CHUNK_SIZE),
    )
  }
}

async function loadContent(bridge: EvenAppBridge, id: string): Promise<string> {
  const nStr = await bridge.getLocalStorage(`${PREFIX}${id}_n`)
  if (!nStr) return ''
  const n = parseInt(nStr, 10)
  let result = ''
  for (let i = 0; i < n; i++) {
    result += await bridge.getLocalStorage(`${PREFIX}${id}_${i}`)
  }
  return result
}
```

### 書き込みのデバウンス

`setLocalStorage` はBLE回線を共有します。頻繁な書き込みはUIの描画を妨げるため、デバウンスしてください。

```typescript
let saveTimer: ReturnType<typeof setTimeout> | null = null

function scheduleSave(state: AppState) {
  if (saveTimer) clearTimeout(saveTimer)
  saveTimer = setTimeout(() => {
    bridge.setLocalStorage('app.state', JSON.stringify(state))
  }, 500)
}
```

## SDKが提供しないもの

以下の機能はEven Hub SDKでは利用できません:

- 直接Bluetoothアクセス
- 任意のピクセル描画（キャンバス以外）
- 音声出力（G2にスピーカーはない）
- テキスト配置の制御
- フォント制御（サイズ・太さ・種類）
- 背景色
- アイテムごとのリストスタイル
- スクロール位置の制御
- アニメーション
- カメラ（G2にはない）
- カラー画像（グレースケールのみ）

## 終了時のクリーンアップ

ハードウェア機能は必ず停止し、イベントリスナーを解除してください。

```typescript
function cleanup() {
  bridge.audioControl(false)
  bridge.imuControl(false)
  unsubscribeEvents()
}

// ライフサイクルイベントで呼ぶ
bridge.onEvenHubEvent(event => {
  const type = event.sysEvent?.eventType ?? 0
  if (type === 6 || type === 7) {  // ABNORMAL_EXIT or SYSTEM_EXIT
    cleanup()
  }
})
```
