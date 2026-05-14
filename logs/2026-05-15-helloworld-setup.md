# 開発ログ: Hello World セットアップ（2026-05-15）

環境: Windows 11、PowerShell、Node.js v22、G2実機あり

## 目標

`D:\Coding\evenhub\myapptest\helloworld` に Hello World アプリを作成し、シミュレーターで表示する。

## 手順と発生した問題

### 1. プロジェクト作成

```bash
npm create vite@latest helloworld -- --template vanilla-ts
cd helloworld && npm install
npm install @evenrealities/even_hub_sdk
```

→ 問題なし。

---

### 2. CLI・シミュレーターのインストール

```bash
npm install -D @evenrealities/evenhub-cli @evenrealities/evenhub-simulator
```

**❌ エラー:**

```
npm error ERESOLVE unable to resolve dependency tree
npm error peer typescript@"^5" from @evenrealities/evenhub-cli
npm error Found: typescript@6.0.3
```

**原因:** `vite@latest` が TypeScript 6.x をインストールするが、`evenhub-cli` は TypeScript ^5 しか対応していない。

**解決策:**

```bash
npm install -D @evenrealities/evenhub-cli @evenrealities/evenhub-simulator --legacy-peer-deps
```

`--legacy-peer-deps` で peer dependency の競合を無視してインストール。動作に支障はなし。

---

### 3. app.json の修正

`npx evenhub init` で生成された `app.json` のデフォルト値に注意：

- `min_sdk_version` が `"0.0.7"` になっている → `"0.0.10"` に修正
- `permissions` に不要なサンプル（network / location）が入っている → `[]` に修正

---

### 4. シミュレーター起動 — PowerShell の罠（3連発）

#### 罠①: PowerShell のスクリプト実行ポリシー

```
'evenhub-simulator' は、内部コマンドまたは外部コマンドとして認識されません
```

**原因:** PowerShell のデフォルト実行ポリシーが `Restricted` のため `.ps1` スクリプトが動かない。

**解決策（1回だけでOK）:**

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

---

#### 罠②: パッケージ名のスコープ忘れ

```bash
npx evenhub-simulator http://localhost:5173
# → npm error 404 Not Found: evenhub-simulator
```

**原因:** スコープ付きパッケージ名を省略した。

**正しいコマンド:**

```bash
npx @evenrealities/evenhub-simulator http://localhost:5173
```

---

#### 罠③: `--legacy-peer-deps` インストール時に `.bin` シムが作られない

罠①②を解決しても、`npx @evenrealities/evenhub-simulator` でシミュレーターが起動しなかった。`node_modules\.bin\` を確認すると `evenhub-simulator` の `.ps1` / `.cmd` が存在しなかった。

**原因:** `--legacy-peer-deps` でインストールした場合、一部環境で `.bin` へのシムリンクが作られないことがある。

**回避策（確実に動く）:**

```powershell
node node_modules/@evenrealities/evenhub-simulator/bin/index.js http://localhost:5173
```

package.json の `scripts` に登録しておくと便利：

```json
{
  "scripts": {
    "dev": "vite",
    "simulator": "node node_modules/@evenrealities/evenhub-simulator/bin/index.js http://localhost:5173"
  }
}
```

以後は `npm run simulator` で起動できる。

---

## 結果

シミュレーターの「Glasses Display」ウィンドウに **「Hello, G2! こんにちは！」** が緑色で表示されることを確認。

## 得られた知見（公式ドキュメントに反映すべき項目）

| 項目 | 内容 | 反映先 |
|---|---|---|
| TypeScript 6.x 競合 | `--legacy-peer-deps` で回避 | 01-quickstart.md |
| PowerShell 実行ポリシー | `Set-ExecutionPolicy RemoteSigned -Scope CurrentUser` | 01-quickstart.md |
| シミュレーター起動コマンド | `npx @evenrealities/evenhub-simulator`（スコープ必須） | 01-quickstart.md |
| `.bin` シム未作成の回避 | `node node_modules/.../bin/index.js` or `npm run simulator` | 01-quickstart.md |
| `app.json` デフォルト値 | `min_sdk_version` と `permissions` を必ず修正 | 01-quickstart.md |
