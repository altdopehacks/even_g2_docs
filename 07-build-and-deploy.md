# ビルドとデプロイ

G2アプリを `.ehpk` パッケージにまとめ、Even Hubへ公開する手順。

## ワークフロー概要

```
app.json 検証 → npm run build → evenhub pack → .ehpk ファイル → Even Hub 開発者ポータル申請
```

## ステップ 1: app.json の検証

公開前に全フィールドを確認します。

```json
{
  "package_id": "com.example.myapp",
  "edition": "202601",
  "name": "My App",
  "version": "1.0.0",
  "min_app_version": "2.0.0",
  "min_sdk_version": "0.0.10",
  "entrypoint": "index.html",
  "permissions": [],
  "supported_languages": ["ja", "en"]
}
```

### フィールド検証ルール

| フィールド | 型 | 必須 | 検証ルール |
|---|---|---|---|
| `package_id` | string | ✓ | 逆ドメイン形式。小文字英数字とドットのみ（ハイフン・大文字・アンダースコア不可）。最低2セグメント |
| `edition` | string | ✓ | `"202601"` 固定 |
| `name` | string | ✓ | 最大**20文字** |
| `version` | string | ✓ | `x.y.z` 形式（`v` プレフィックス不可）|
| `min_app_version` | string | ✓ | `"2.0.0"` 推奨 |
| `min_sdk_version` | string | ✓ | `"0.0.10"` 以上 |
| `entrypoint` | string | ✓ | ビルド出力フォルダ内に存在するファイル |
| `permissions` | array | ✓ | オブジェクトの配列（空 `[]` も可） |
| `supported_languages` | array | ✓ | 対応言語コードの配列 |

### 権限の設定方法

```json
"permissions": [
  {
    "name": "network",
    "desc": "天気データをAPIから取得します",
    "whitelist": ["https://api.weather.com"]
  },
  {
    "name": "g2-microphone",
    "desc": "ハンズフリー操作のための音声コマンド"
  }
]
```

**利用可能な権限名:**
- `network` — 外部ネットワークアクセス（`whitelist` 必須）
- `location` — GPS/位置情報
- `g2-microphone` — G2グラスのマイク
- `phone-microphone` — ペアリングしたスマートフォンのマイク
- `album` — フォトアルバムアクセス
- `camera` — カメラアクセス

## ステップ 2: ビルド

```bash
npm run build
```

`dist/` フォルダが作成され、ファイルが存在することを確認してください。

## ステップ 3: パッケージング

```bash
npx evenhub pack app.json dist -o my-app.ehpk
```

オプション:

| オプション | 説明 |
|---|---|
| `-o <ファイル名>` / `--output <ファイル名>` | 出力ファイル名（デフォルト: `out.ehpk`）|
| `--no-ignore` | ドットファイルを含める |
| `-c` / `--check` | `package_id` がEven Hubストアで利用可能か確認 |

## ステップ 4: 出力の確認

```bash
ls -lh my-app.ehpk
```

`.ehpk` ファイルが生成されていることを確認します。

## ステップ 5: Even Hub 開発者ポータルへ申請

生成した `.ehpk` ファイルをEven Hub開発者ポータルに提出します。審査・互換性チェック後、G2ユーザーに公開されます。

## CloudFront Functions による APIキー Edge 注入（本番推奨）

APIキーをクライアント（WebView）に露出させたくない場合、**AWS CloudFront Functions でリクエストにキーを動的注入**する構成が有効です。Even Hub の `app.json` whitelist も1ドメイン（CloudFrontのドメイン）に集約できます。

### 構成

```
WebView → CloudFront（1ドメイン）
              ├── /api/*    → api.example.com（APIキー注入）
              ├── /static/* → S3（SPA本体）
              └── /tiles/*  → 外部データソース
```

### CloudFront Function（APIキー注入）

```javascript
// CloudFront Function: viewer-request イベント
function handler(event) {
  var request = event.request
  // エッジでAPIキーをクエリパラメータに追加
  request.querystring['apiKey'] = { value: 'YOUR_API_KEY' }
  return request
}
```

### Terraform でのキー管理

```terraform
# Secrets Manager からキーを取得してFunctionに埋め込む
data "aws_secretsmanager_secret_version" "api_key" {
  secret_id = "myapp/api-key"
}

resource "aws_cloudfront_function" "api_proxy" {
  code = templatefile("cf-function.js", {
    api_key = data.aws_secretsmanager_secret_version.api_key.secret_string
  })
}
```

キーローテーション時は Secrets Manager 更新後に `terraform apply` を再実行するだけで1〜2分で全エッジに反映されます。

### app.json の whitelist

```json
"permissions": [
  {
    "name": "network",
    "desc": "データ取得",
    "whitelist": ["https://your-cloudfront-domain.cloudfront.net"]
  }
]
```

CloudFront を一枚挟むことで、複数のAPIを使う場合も whitelist が1エントリで済みます。

## CORS の注意点

Even AppはChromium（Android）またはWKWebView（iOS）のブラウザエンジンで動作するため、**完全なCORSが適用されます**。

`app.json` の `whitelist` はEven側の権限チェックであり、CORSをバイパスしません。以下の両方が必要です:

1. `app.json` の `permissions.network.whitelist` にドメインを追加
2. 呼び出すAPIが正しいCORSヘッダー（`Access-Control-Allow-Origin` など）を返す

| シナリオ | 対処法 |
|---|---|
| APIにCORSヘッダーがある | `app.json` にドメインをホワイトリスト登録するだけでOK |
| APIにCORSヘッダーがない | 自前バックエンド、CloudflareWorkerプロキシ、またはCORS対応のミラーを使用 |
| 開発中のlocalhostがCORSでブロックされる | `vite.config.ts` にViteプロキシを設定 |

### Vite プロキシ設定（開発用）

```typescript
// vite.config.ts
export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: 'https://api.example.com',
        changeOrigin: true,
        rewrite: (p) => p.replace(/^\/api/, ''),
      },
    },
  },
})
```

> このプロキシは開発時のみ有効です。本番の `.ehpk` ではWebViewが直接リクエストを送るため、APIにCORSヘッダーが必要です。

## トラブルシューティング

| エラー | 対処法 |
|---|---|
| `Invalid package id` | 逆ドメイン形式で最低2セグメント。小文字英数字のみ（ハイフン・大文字・数字始まり不可） |
| `name: must be 20 characters or fewer` | `name` フィールドを20文字以内に短縮 |
| `version: must be in x.y.z format` | `"1.0.0"` のような3部構成の数値バージョンを使用 |
| `min_app_version: expected string, received undefined` | `min_app_version` フィールドを追加（例: `"2.0.0"`） |
| `min_sdk_version: expected string, received undefined` | `min_sdk_version` フィールドを追加（例: `"0.0.10"`） |
| `permissions: each permission must be an object...` | `permissions` はオブジェクト配列。キー・バリュー形式は不可 |
| `supported_languages: invalid language` | 対応言語コードのみ使用: `en`, `de`, `fr`, `es`, `it`, `zh`, `ja`, `ko` |
| `Entrypoint file not found` | `npm run build` 後にビルド出力フォルダ内に `entrypoint` のファイルが存在するか確認 |
| `Project folder not found` | `npm run build` を先に実行してから再試行 |

## シミュレーターの既知の制限

実機では動くがシミュレーター（v0.7.1）では動かない制限があります。

| 制限 | シミュレーター | 実機（SDK 0.0.10+） |
|---|---|---|
| コンテナ数 | **4個まで**（5個以上はリジェクト） | 最大12個 |
| 画像コンテナサイズ | **200×100 px まで** | 仕様通り（最大288×144） |

> シミュレーターでエラーになっても、実機では動く可能性があります。最終確認は必ず実機で行ってください。

## バージョン管理のベストプラクティス

```json
{
  "version": "1.0.0"  // 初回リリース
  "version": "1.0.1"  // バグ修正
  "version": "1.1.0"  // 機能追加
  "version": "2.0.0"  // 破壊的変更
}
```

Even Hub ストアに申請するたびにバージョンを上げてください。

## Claude Code スキルでの実行

```bash
/build-and-deploy
```

上記スキルが app.json の検証・ビルド・パッキングを自動化します。
