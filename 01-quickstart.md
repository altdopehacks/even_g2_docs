# クイックスタートガイド

Even Realities G2向けアプリの作成から、シミュレーターでの動作確認・実機テストまでの手順。

## 動作環境

> **注意**: G2のコンパニオンアプリは **iPhone（iOS）のみ**対応です。Androidは現時点では非対応です。

アーキテクチャの実態:

```
[あなたのサーバー] <--HTTPS--> [iPhoneのWebView] <--BLE 5.x--> [G2グラス]
```

- WebViewの実装は **`flutter_inappwebview`**（実際のブラウザエンジン）
- iPhoneはBLEプロキシとして動くだけ。アプリロジックはサーバー上で動く
- 開発中はPCのdev serverをサーバーとして使い、本番はAWSなど任意のホスティングに移行できる

## 前提条件

- Node.js v18 以上
- Claude Code インストール済み・認証済み
- **iPhone**（Even Hub コンパニオンアプリをインストール）

## 1. プロジェクト作成

Claude Code で以下のスキルを実行するか、手動でセットアップします。

```bash
/quickstart my-app
```

### 公式テンプレートを使う（推奨）

`evenhub-templates` リポジトリに4種類のスターターテンプレートがあります。`degit` で ZIP ダウンロード（git履歴なし）できます。

```bash
# 最小構成（Vanilla TS）
npx degit even-realities/evenhub-templates/minimal my-app

# マルチページ構成
npx degit even-realities/evenhub-templates/multipage my-app

# 音声認識（ASR）テンプレート
npx degit even-realities/evenhub-templates/asr my-app

# テキスト大量表示（ページネーション）テンプレート
npx degit even-realities/evenhub-templates/text-heavy my-app
```

テンプレートのインストール後:

```bash
cd my-app
npm install
```

#### テンプレート選択の目安

| テンプレート | 用途 |
|---|---|
| `minimal` | シンプルなUIアプリ。まず触るならこれ |
| `multipage` | 画面遷移が必要なアプリ |
| `asr` | マイク入力 + STT（音声認識）アプリ。STTプロバイダーは自分で差し替え |
| `text-heavy` | 長文記事・ニュース表示。`@evenrealities/pretext` でページネーション |

#### ASR テンプレートの構造

```
asr/
├── src/
│   ├── main.ts         # UIとイベントループ
│   └── asr/
│       └── stt.ts      # STTスタブ（← ここにSTTプロバイダーを実装）
```

`stt.ts` のインターフェース:

```typescript
export interface SttClient {
  sendPcm(chunk: Uint8Array): void  // 10ms・40byteのPCMチャンクを渡す
  close(): void
}

// 自分のSTTサービスに合わせてここを実装する
export function startSttStream(
  apiKey: string,
  onSnapshot: (text: string) => void,
  onError: (err: Error) => void,
): SttClient {
  throw new Error('STT provider not implemented')
}
```

### 手動セットアップ

```bash
# Vite + TypeScript プロジェクト作成
npm create vite@latest my-app -- --template vanilla-ts
cd my-app && npm install

# SDK インストール
npm install @evenrealities/even_hub_sdk

# 開発ツール（CLI・シミュレーター）インストール
npm install -D @evenrealities/evenhub-cli @evenrealities/evenhub-simulator

# app.json マニフェスト生成
npx evenhub init
```

## 2. app.json の設定

プロジェクトルートに `app.json` を配置します。

```json
{
  "package_id": "com.example.myapp",
  "edition": "202601",
  "name": "My App",
  "version": "0.1.0",
  "min_app_version": "2.0.0",
  "min_sdk_version": "0.0.10",
  "entrypoint": "index.html",
  "permissions": [],
  "supported_languages": ["ja", "en"]
}
```

### フィールド説明

| フィールド | 説明 | 注意点 |
|---|---|---|
| `package_id` | 逆ドメイン形式の一意ID | ハイフン不可。小文字英数字とドットのみ |
| `edition` | プラットフォームエディション | `"202601"` 固定 |
| `name` | アプリ表示名 | 最大20文字 |
| `version` | セマンティックバージョン | `x.y.z` 形式 |
| `min_app_version` | 必要な最小Even Hubアプリバージョン | `"2.0.0"` 推奨 |
| `min_sdk_version` | 必要な最小SDKバージョン | `"0.0.10"` 以上 |
| `entrypoint` | エントリーポイントHTMLファイル | `"index.html"` |
| `permissions` | 権限配列 | 不要なら `[]` |
| `supported_languages` | 対応言語コード配列 | `ja`, `en` など |

## 3. src/main.ts の実装

`src/main.ts` を以下の内容で置き換えます（Hello World）。

```typescript
import {
  waitForEvenAppBridge,
  TextContainerProperty,
  CreateStartUpPageContainer,
} from '@evenrealities/even_hub_sdk'

const bridge = await waitForEvenAppBridge()

const mainText = new TextContainerProperty({
  xPosition: 0,
  yPosition: 0,
  width: 576,
  height: 288,
  borderWidth: 0,
  borderColor: 5,
  paddingLength: 4,
  containerID: 1,
  containerName: 'main',
  content: 'G2からこんにちは！',
  isEventCapture: 1,
})

const result = await bridge.createStartUpPageContainer(
  new CreateStartUpPageContainer({
    containerTotalNum: 1,
    textObject: [mainText],
  })
)

console.log('ページ作成:', result === 0 ? '成功' : '失敗')
```

## 4. 開発サーバー起動

```bash
npm run dev
```

Vite が `http://localhost:5173` でサーブします（ポートが使用中の場合は別ポートになります）。

## 5. シミュレーターで確認

```bash
npx evenhub-simulator http://localhost:5173
```

シミュレーターはG2ディスプレイ（576×288、4ビットグレースケール）をデスクトップウィンドウで描画します。

### グロウエフェクトを有効にする

実機に近い表示（緑の発光）を確認するには、シミュレーターのUIでグロウエフェクトを有効にします。

## 6. 実機でテスト

PCとG2が同じWi-Fiに接続されている状態で：

```bash
npx evenhub qr --url http://<あなたのIPアドレス>:5173
```

表示されたQRコードをEven Hubコンパニオンアプリでスキャンすると、G2にアプリがサイドロードされます。

### ローカルIPアドレスの確認

```powershell
# Windows
ipconfig
```

`IPv4アドレス` の値（例: `192.168.1.10`）を使います。

## プロジェクト構成（完成形）

```
my-app/
├── app.json          # アプリマニフェスト
├── index.html        # エントリーポイント
├── package.json
├── tsconfig.json
└── src/
    └── main.ts       # アプリロジック
```

## 開発中の既知の制限

### GPS（位置情報）はサイドロード中に動作しない

`navigator.geolocation` はEven HubのWebViewで利用可能ですが、**QRコードサイドロードでの開発中は `PERMISSION_DENIED` が返ります**。Even Hub開発者ポータルに正式アップロードしたアプリとしてインストールした場合のみ機能します。

```typescript
// 開発中（QRサイドロード）→ PERMISSION_DENIED
// 本番（Hub正式インストール）→ 動作する
navigator.geolocation.getCurrentPosition(
  (pos) => console.log(pos.coords),
  (err) => console.error(err.code)  // 開発中は code=1 (PERMISSION_DENIED)
)
```

GPS機能を使うアプリは、実機テストの最終確認を正式インストール後に行ってください。

## 次のステップ

- [UIの構築](03-ui-design.md) — コンテナ・テキスト・リスト・画像の使い方
- [入力処理](04-input-handling.md) — タッチパッドジェスチャーのハンドリング
- [ビルドと公開](07-build-and-deploy.md) — `.ehpk` パッケージングと申請
