# 低レベルBLEプロトコル（上級者向け）

Even Hub SDK が内部で使用するBLEプロトコルのリファレンス。通常アプリ開発では不要ですが、SDK非経由でG2を直接操作したい場合や、カスタムファームウェア・サードパーティSDK開発の参考になります。

> **注意**: 以下の情報はコミュニティによるリバースエンジニアリングとOSSコード解析（[i-soxi/even-g2-protocol](https://github.com/i-soxi/even-g2-protocol)・[EvenDemoApp](https://github.com/even-realities/EvenDemoApp)・[even-g2-notes](https://github.com/nickustinov/even-g2-notes)）に基づきます。公式仕様ではなく、変更される可能性があります。

## BLE接続の構造

G2には**左アーム・右アームそれぞれ独立したBLE接続**があります。

```
任意のホスト（iPhone / PC / Mac / Raspberry Pi など）
  ├── BLE接続 (左アーム)  ←→  G2 左アーム  Even G2_XX_L_YYYYYY
  └── BLE接続 (右アーム)  ←→  G2 右アーム  Even G2_XX_R_YYYYYY
```

- 主な制御（ディスプレイ・マイク）は**右アーム**経由
- 左アームはミラーリング・冗長系として機能
- **標準BLEペアリング不要** — カスタム7パケット認証ハンドシェイクのみ
- **iPhoneもEven Hub SDKも必須ではない** — PCや任意のBLE対応デバイスから直接接続可能

## BLE サービス・Characteristics

```
メインサービス UUID: 00002760-08c2-11e1-9073-0e8ac72e0000

  Characteristic 5401  Write Without Response  phone → glasses コマンド送信
  Characteristic 5402  Notify                  glasses → phone レスポンス・イベント受信
  Characteristic 6402  Write Without Response  phone → glasses 描画コマンド（Rendering Channel）
```

| Characteristic | 方向 | 用途 |
|---|---|---|
| `5401` | 送信 | コマンド（Content Channel） |
| `5402` | 受信 | レスポンス・タッチイベント |
| `6402` | 送信 | 描画コマンド（Rendering Channel） |

MTU: **512バイト**（マルチパケット分割対応）

## パケット構造

```
[AA] [type] [seq] [len] [pkt_total] [pkt_serial] [svc_hi] [svc_lo] [payload...] [crc_lo] [crc_hi]
  0     1     2     3       4            5            6        7        8...          n-1     n
```

| オフセット | フィールド | 値・説明 |
|---|---|---|
| 0 | Magic | 常に `0xAA` |
| 1 | Type | `0x21` = コマンド（phone→glasses）/ `0x12` = レスポンス（glasses→phone） |
| 2 | Sequence | 0〜255 のインクリメントカウンタ |
| 3 | Length | ペイロード長 + 2（CRC2バイト含む） |
| 4 | Packet Total | このメッセージの総パケット数 |
| 5 | Packet Serial | このパケットの番号（0始まり） |
| 6–7 | Service ID | サービス識別子（2バイト、big-endian） |
| 8… | Payload | Protobuf varint 符号化 |
| n-1, n | CRC | CRC-16/CCITT リトルエンディアン |

### CRC計算

```typescript
// CRC-16/CCITT — ペイロード部分のみ（8バイトヘッダーは除く）
function crc16ccitt(payload: Uint8Array): number {
  let crc = 0xFFFF
  for (const byte of payload) {
    crc ^= byte << 8
    for (let i = 0; i < 8; i++) {
      crc = (crc & 0x8000) ? ((crc << 1) ^ 0x1021) : (crc << 1)
    }
    crc &= 0xFFFF
  }
  return crc
}
// 結果をリトルエンディアンで末尾に追加: [crc & 0xFF, crc >> 8]
```

### マルチパケット送信

MTU（512バイト）を超えるデータは分割します。

```
パケット1: [AA][21][seq][len][3][0][svc_hi][svc_lo][payload part1][crc]
パケット2: [AA][21][seq][len][3][1][svc_hi][svc_lo][payload part2][crc]
パケット3: [AA][21][seq][len][3][2][svc_hi][svc_lo][payload part3][crc]
```

## 認証ハンドシェイク

接続後、最初に7パケットのハンドシェイクが必要です。PINは不要です。

```
1. phone → glasses: [0x80-00] セッション開始（タイムスタンプ・トランザクションID）
2. glasses → phone: [0x80-01] 認証チャレンジ
3. phone → glasses: [0x80-20] 認証レスポンス（ペイロード付き）
   ... 計7パケット
```

認証完了後にディスプレイコマンドやテレプロンプターが使用可能になります。

## サービスID一覧

| Service ID | 名前 | 説明 |
|---|---|---|
| `0x80-00` | SESSION | セッション管理・同期 |
| `0x80-20` | AUTH | ペイロード付き認証 |
| `0x80-01` | AUTH_RESP | グラス認証応答 |
| `0x04-20` | DISPLAY_INIT | ディスプレイ起動 |
| `0x06-20` | TELEPROMPTER | テレプロンプター（テキスト表示） |
| `0x0B-20` | ASR | 音声文字変換 |
| `0x0C-20` | TASK | タスク/Todo機能 |
| `0x0D-00` | DEVICE_SETTINGS | デバイス設定 |
| `0x0E-20` | DISPLAY_CONFIG | 画面構成設定 |

## テレプロンプター（Service 0x06-20）

スクロール可能なテキストをG2に表示する機能。動作確認済み。

### 送信フロー

```
1. 認証ハンドシェイク（7パケット）
2. ディスプレイ設定         [0x0E-20, type=2]  幅267・行高230など
3. スクリプト初期化         [0x06-20, type=1]  スクリプト番号・モード指定
4. コンテンツページ 0〜9    [0x06-20, type=3]  最初のバッチ（10ページ）
5. 中間マーカー             [0x06-20, type=255] 必須の区切りシグナル
6. コンテンツページ 10〜11  [0x06-20, type=3]  第2バッチ
7. 同期トリガー             [0x80-00, type=14] レンダリング実行
8. 残りのページ             [0x06-20, type=3]  以降のページ
```

### テキストレイアウト仕様

| 項目 | 値 |
|---|---|
| 1行あたりの文字数 | 約25文字（自動折り返し） |
| 1ページあたりの行数 | 10行 |
| 同時表示行数 | 約7行 |
| スクロールモード | `0x00` = 手動（スワイプ）/ `0x01` = AI自動 |

### 最小コンテンツ要件

- 約10ページ未満では表示されない可能性あり
- スクリプトインデックスは既存のもの（0または1）を参照

## EvenDemoApp コマンド一覧（高レベルプロトコル）

EvenDemoApp で使われるコマンドバイト（Even Hub SDK が内部で使用する形式）:

| コマンドバイト | 名前 | 方向 | 説明 |
|---|---|---|---|
| `0x0E` | MIC_CONTROL | 送信 | マイクON/OFF。`0x0E 0x01` でON、`0x0E 0x00` でOFF |
| `0xF1` | MIC_DATA | 受信 | LC3エンコードされた音声データ（フレーム単位） |
| `0xF5` | TOUCH_EVENT | 受信 | タッチパッド・リングからのジェスチャーイベント |
| `0x4E` | AI_RESULT | 送信 | テキスト（AI応答など）をページ単位でG2に表示 |
| `0x15` | BMP_PACKET | 送信 | 4ビットグレースケール画像データ（194バイト/パケット） |
| `0x16` | BMP_CRC | 送信 | 画像転送のCRC32（XZ多項式）チェックサム |
| `0x20 0x0D 0x0E` | BMP_END | 送信 | 画像転送の終端シーケンス |

## マイク（音声）フロー

```
G2グラス                     iPhone                          アプリ(WebView)
   │  長押し検出               │                                   │
   │──[0xF5 0x17]────────────>│                                   │
   │                          │── audioControl(true) ──────────>  │
   │<─[0x0E 0x01]─────────────│                                   │
   │  LC3音声ストリーム         │                                   │
   │──[0xF1 ...frames...]────>│  LC3デコード → PCM S16LE 16kHz    │
   │                          │──── audioEvent.audioPcm ─────────>│
   │                          │                                   │  STT処理
   │                          │<── audioControl(false) ──────────│
   │──[0x0E 0x00]────────────>│                                   │
```

### LC3コーデック仕様

| 項目 | 値 |
|---|---|
| コーデック | LC3（Bluetooth LE Audio） |
| サンプリングレート | 16kHz |
| チャンネル | モノラル |
| デコード後フォーマット | PCM S16LE（符号付き16ビットリトルエンディアン） |
| フレーム長 | 10ms |
| フレームサイズ（デコード後） | 40バイト |

EvenDemoApp では `liblc3`（LC3デコーダー）と `rnnoise`（ノイズリダクション）を使用しています。

## タッチイベント（0xF5）

```
受信: [0xF5][イベント種別]
```

| イベント種別バイト | ジェスチャー |
|---|---|
| `0x00` | シングルタップ |
| `0x01` | ダブルタップ |
| `0x02` | 上スワイプ |
| `0x03` | 下スワイプ |
| `0x17` | 長押し（マイク起動トリガー） |

## テキスト表示フロー（0x4E）

長いテキストは複数ページに分割して送信します。

```
送信: [0x4E][ページ番号][合計ページ数][テキストデータ...]
```

- ページ番号: 0始まり
- 1ページのテキスト上限: 約400〜500文字
- G2ファームウェアがページング表示を管理

## 画像転送フロー（0x15 / 0x16）

4ビットグレースケールのBMPデータを分割転送します。

```
1. 画像データを194バイト/パケットに分割
2. 各パケットを [0x15][シーケンス番号][データ...] で送信
3. 全パケット送信後、[0x20 0x0D 0x0E] で終端を通知
4. [0x16][CRC32XZ下位][CRC32XZ上位] でチェックサムを送信
```

CRC多項式: CRC32-XZ（`0xEDB88320`、逆ビット順）

## AIフロー（EvenDemoApp実装例）

```
1. G2長押し → [0xF5 0x17]
2. アプリ → [0x0E 0x01]（マイクON）
3. G2 → [0xF1 ...] × N フレーム（LC3音声）
4. アプリがLC3をデコード → PCM
5. PCM → WhisperまたはDeepSeek STT → テキスト
6. テキスト → DeepSeek LLM → 応答テキスト
7. 応答テキストをページ分割 → [0x4E] × ページ数 で送信
8. G2にテキスト表示
```

## 直接BLE接続（SDK非経由）の用途

iPhoneとEven Hub SDKを使わずに直接G2を操作できます。

| 用途 | 説明 |
|---|---|
| PCからG2を操作 | WindowsやMacのPythonスクリプト等からBLE接続 |
| カスタムホストデバイス | Raspberry Piなど任意のBLE対応デバイスをホストにする |
| テレプロンプター | スライド発表・台本読み上げ用途（動作確認済み） |
| プロトタイピング | Even Hub SDK外での独自機能開発 |

```python
# Pythonでの直接接続例（bleak ライブラリ）
from bleak import BleakClient

WRITE_CHAR  = "00005401-08c2-11e1-9073-0e8ac72e0000"
NOTIFY_CHAR = "00005402-08c2-11e1-9073-0e8ac72e0000"

async def connect_g2(address: str):
    async with BleakClient(address) as client:
        await client.start_notify(NOTIFY_CHAR, on_notify)
        await authenticate(client)   # 7パケットハンドシェイク
        await send_teleprompter(client, text)
```

## 参考リポジトリ

| リポジトリ | 内容 |
|---|---|
| [i-soxi/even-g2-protocol](https://github.com/i-soxi/even-g2-protocol) | BLEプロトコルのリバースエンジニアリング。パケット構造・認証・テレプロンプターの動作実装あり |
| [even-realities/EvenDemoApp](https://github.com/even-realities/EvenDemoApp) | Flutter製BLEデモ。LC3デコード・AI連携の実装例 |
| [nickustinov/even-g2-notes](https://github.com/nickustinov/even-g2-notes) | コミュニティによるプロトコル解析メモ |
| [BxNxM/even-dev](https://github.com/BxNxM/even-dev) | Python/ESPによる直接BLE制御の実験 |
