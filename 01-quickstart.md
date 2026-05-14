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

### even-toolkit でスキャフォールディング（React + デザインシステム付き）

スマホ側設定UIとグラス表示を両方作る場合は **even-toolkit** の `create-even-app` が最速です。

```bash
npx @even-toolkit/create-even-app my-app
```

6種類のテンプレートから選択できます：

| テンプレート | 用途 |
|---|---|
| `minimal` | 最小構成 |
| `dashboard` | データ表示・ウィジェット中心 |
| `notes` | テキスト入力・メモアプリ |
| `chat` | チャット・会話UI |
| `tracker` | 記録・トラッキング |
| `media` | 画像・メディア表示 |

### evenhub-templates を使う（Vanilla TS）

React不要でシンプルに始めたい場合は `evenhub-templates` を使います。`degit` で ZIP ダウンロード（git履歴なし）できます。

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

#### evenhub-templates 選択の目安

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
# ※ vite@latest は TypeScript 6.x をインストールするが evenhub-cli は ^5 要求のため
#   --legacy-peer-deps が必要（動作には影響なし）
npm install -D @evenrealities/evenhub-cli @evenrealities/evenhub-simulator --legacy-peer-deps

# app.json マニフェスト生成
npx evenhub init
```

## 2. app.json の設定

`npx evenhub init` で `app.json` を生成できます。ただし**デフォルト値に2点注意**してください：

- `min_sdk_version` が古い値（`"0.0.7"` など）になっている → `"0.0.10"` に修正
- `permissions` にサンプルの network / location が入っている → 不要なら `[]` に修正

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
node node_modules/@evenrealities/evenhub-simulator/bin/index.js http://localhost:5173
```

`package.json` の `scripts` に登録しておくと毎回楽です：

```json
{
  "scripts": {
    "dev": "vite",
    "simulator": "node node_modules/@evenrealities/evenhub-simulator/bin/index.js http://localhost:5173"
  }
}
```

以後は `npm run simulator` で起動できます。

シミュレーターはG2ディスプレイ（576×288、4ビットグレースケール）をデスクトップウィンドウで描画します。

### Windows で起動しない場合

**PowerShell のスクリプト実行ポリシーエラー**が出た場合は1回だけ以下を実行：

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

**`--legacy-peer-deps` でインストールした場合に `.bin` シムが作られないことがある。** その場合は直接 `node` で起動するか、`package.json` の `scripts` に登録する：

```powershell
# 直接起動
node node_modules/@evenrealities/evenhub-simulator/bin/index.js http://localhost:5173
```

```json
// package.json に登録しておくと便利
{
  "scripts": {
    "dev": "vite",
    "simulator": "node node_modules/@evenrealities/evenhub-simulator/bin/index.js http://localhost:5173"
  }
}
```

登録後は `npm run simulator` で起動できる。

### グロウエフェクトを有効にする

実機に近い表示（緑の発光）を確認するには、シミュレーターのUIでグロウエフェクトを有効にします。

### マイク入力をシミュレーターでテストする（ASR開発）

`AUDIO_DEVICE` 環境変数でシミュレーターに使用するオーディオ入力デバイスを指定できます。`--aid` フラグとして渡され、G2マイク相当の音声入力でASRをテストできます。

```bash
# macOS / Linux
AUDIO_DEVICE="Built-in Microphone" npx evenhub-simulator http://localhost:5173

# Windows（PowerShell）
$env:AUDIO_DEVICE="マイク (Realtek Audio)"; npx evenhub-simulator http://localhost:5173
```

利用可能なデバイス名はOS のサウンド設定で確認してください。デバイス名を省略した場合はデフォルトのマイクが使用されます。

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
