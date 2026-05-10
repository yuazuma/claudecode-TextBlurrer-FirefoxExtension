# プロンプトと応答の記録

---

## ターン1

**ユーザー:**

Chrome拡張機能であるTextBlurrer-ChromeExtension ( https://github.com/horihiro/TextBlurrer-ChromeExtension ) について、Firefox拡張機能に移植したいと思います。実現可能性と実装案を提示してください。

---

**アシスタント:**

Chrome拡張機能の構成を調査した結果、以下の通りです。

### 実現可能性

**高い。** ほぼそのまま移植できます。

TextBlurrer-ChromeExtension が使っているAPIは以下の通りで、すべてFirefoxで動作します：

| Chrome API | Firefox での対応 |
|---|---|
| `chrome.storage.local` | 完全対応 (`chrome.*` 名前空間をエイリアスとして利用可) |
| `chrome.contextMenus` | `browser.menus` として対応（`chrome.contextMenus` エイリアスも有効） |
| `chrome.runtime.*` | 完全対応 |
| `chrome.tabs.*` | 完全対応 |
| `MutationObserver`, Shadow DOM | 標準Web API、完全対応 |
| CSS `filter: blur()` | 完全対応 |
| CSS `mask` / `-webkit-mask` | `-moz-` プレフィックス追加が必要な場合あり |

### 必要な変更点

#### 1. `manifest.json` の修正

最大の差分はここです。

```jsonc
// Chrome (MV3)
{
  "manifest_version": 3,
  "background": {
    "type": "module",         // ← Firefox 128未満では非対応
    "service_worker": "dist/background/service-worker.js"
  }
}

// Firefox (MV3) に追加・変更が必要な箇所
{
  "manifest_version": 3,
  "browser_specific_settings": {        // ← Firefox必須 (AMO登録に必要)
    "gecko": {
      "id": "text-blurrer@example.com",
      "strict_min_version": "128.0"
    }
  },
  "background": {
    "scripts": ["dist/background/service-worker.js"]  // ← "type":"module" を除去
    // または Firefox 128+ なら service_worker のままでOK
  }
}
```

#### 2. `chrome.contextMenus` → `browser.menus`

Firefoxでは `chrome.contextMenus` エイリアスが使えるため **コード変更は最小限**。

#### 3. CSS `-webkit-mask` の互換性

`-webkit-mask` に `mask` (標準) を併記する必要があります。

#### 4. ビルド設定の調整

TypeScript + bundler を使っているため、Firefox向けのビルドターゲット設定を追加するだけで対応できます。

### 実装案（3ステップ）

```
Step 1: Chromeソースをベースにソースコードをコピー
Step 2: manifest.json をFirefox向けに修正
Step 3: CSS の webkit プレフィックス対応 + ビルド設定追加
```

### まとめ

変更が必要なファイルは実質 **`manifest.json`（必須）** と **`public/css/main.css`（軽微）** の2ファイルのみです。ロジック部分のTypeScriptコードはほぼ無変更で動きます。

---

## ターン2

**ユーザー:**

実装を進めてください。

---

**アシスタント:**

Chrome拡張機能のソースを取得して、Firefox向けに構築しました。

作成したファイル構成：

```
TextBlurrer-FirefoxExtension/
├── src/                          ← TypeScriptソース（Chrome版から移植）
│   ├── background/background.ts
│   ├── content/
│   │   ├── contentScript.ts
│   │   └── blurrer/{DOMBlurrer,InputBlurrer,TitleBlurrer,IBlurrer}.ts
│   ├── popup/popup.ts
│   └── utils/common.ts
├── public/                       ← ポップアップUI素材
│   ├── main.html
│   ├── css/main.css
│   └── img/*.svg (6ファイル)
├── dist/                         ← ビルド出力（gitignore済み）
│   ├── background/service-worker.js  (704B)
│   ├── contentScript/main.js         (22.6KB)
│   └── popup/js/bundle.js            (7.5KB)
├── manifest.json                 ← Firefox向けに修正済み
├── package.json / tsconfig.json / .gitignore
└── img/                          ← PNG アイコン（手動コピーが必要）
```

Chrome版からの変更点（2箇所のみ）：

| ファイル | 変更内容 |
|---|---|
| `manifest.json` | `browser_specific_settings.gecko` 追加、`"type": "module"` を background から除去 |
| TypeScript全ファイル | **変更なし**（Firefoxは `chrome.*` をエイリアスとしてサポート） |
| CSS | **変更なし**（元々 `-webkit-mask` と `mask` の両方が記述済み） |

手動対応が1つ必要：

```
img/icon16.png   ← Chrome拡張機能リポジトリから手動コピー
img/icon128.png  ← Chrome拡張機能リポジトリから手動コピー
```

Firefoxへの読み込み方法：
1. Firefoxで `about:debugging` を開く
2. 「このFirefox」→「一時的なアドオンを読み込む」
3. このリポジトリの `manifest.json` を選択

---

## ターン3

**ユーザー:**

Firefoxへの読み込み方法を実施したところ、一時的なアドオンのインストール中にエラーがありました。

エラーの詳細：  
`background.service_worker is currently disabled. Add background.scripts.`

---

**アシスタント:**

Firefox のバージョンによっては `background.service_worker` が未サポートで、`background.scripts` が必要です。`manifest.json` を修正しました。

変更内容：

```json
// 変更前
"background": {
  "service_worker": "dist/background/service-worker.js"
},
"browser_specific_settings": {
  "gecko": {
    "strict_min_version": "128.0"
  }
}

// 変更後
"background": {
  "scripts": ["dist/background/service-worker.js"]
},
"browser_specific_settings": {
  "gecko": {
    "strict_min_version": "109.0"
  }
}
```

**原因：** `background.service_worker` はFirefox 128以降でしか動作しません。Firefox 109〜127 では MV3 でも `background.scripts`（イベントページ形式）が必要です。Chrome と異なり、Firefox は MV2 相当の「永続しないバックグラウンドスクリプト」として処理するため、動作上の問題はありません。

---

## ターン4

**ユーザー:**

続いて、警告が表示されました。  
`Reading manifest: Warning processing version_name: An unexpected property was found in the WebExtension manifest.`

---

**アシスタント:**

`version_name` は Chrome 固有のプロパティで Firefox では未対応です。`manifest.json` から削除しました。

```json
// 変更前
"version": "0.2.6",
"version_name": "0.2.6",

// 変更後
"version": "0.2.6",
```

---

## ターン5

**ユーザー:**

パッケージを作成してhttps://addons.mozilla.orgで配布できるようにする手順を教えてください。

---

**アシスタント:**

AMO配布には **web-ext**（Mozilla公式CLI）でパッケージを作り、Developer Hubから申請するのが標準手順です。

`package.json` に `web-ext` を追加し、`sign` スクリプトも追加しました：

```json
"scripts": {
  "package": "npm run build && web-ext build --source-dir . ...",
  "sign": "web-ext sign --source-dir . --artifacts-dir ./web-ext-artifacts --channel unlisted ..."
}
```

### AMO配布の完全手順

#### 1. ZIPパッケージを作成

```bash
npm run package
```

`web-ext-artifacts/text_blurrer-0.2.6.zip` が生成されます。

#### 2. ソースコードZIPも別途作成

AMOではminify/bundleされた拡張機能には**ソースコードの提出が必須**です。

```bash
git archive --format=zip HEAD -o source-code.zip
```

#### 3. AMO Developer Hub で申請

1. [addons.mozilla.org/developers](https://addons.mozilla.org/developers/) にログイン
2. 「新しいアドオンを申請」をクリック
3. **Distribution method** → `On this site` を選択
4. ZIPをアップロード
5. ソースコードZIPをアップロード
6. メタ情報を入力して申請

#### セルフ配布（AMO審査なし）

```bash
npm run sign -- --api-key=<KEY> --api-secret=<SECRET>
```

---

## ターン6

**ユーザー:**

AMO Developer Hub で申請する前の手順まで実行して、AMO Developer Hubには公開せずにセルフ配布（AMO審査なし）できるように.xpiを生成する手順を実行してください。

---

**アシスタント:**

まず `npm run package` でZIPを生成しました：

```
> Building web extension from .
> Your web extension is ready: web-ext-artifacts\text_blurrer-0.2.6.zip
```

次に `web-ext sign` でXPIを作成するためにAMO APIキーが必要であることを確認し、APIキーの取得手順を案内しました。

APIキーの取得先：  
https://addons.mozilla.org/developers/addon/api/key/

| 項目 | 用途 |
|---|---|
| JWT issuer | `--api-key` に指定 |
| JWT secret | `--api-secret` に指定 |

取得後のコマンド：

```bash
npm run sign -- --api-key=<KEY> --api-secret=<SECRET>
```

---

## ターン7

**ユーザー:**

```
npm run sign -- --api-key=user:XXXXXXXX:XXX --api-secret=<SECRET_MASKED>
```
を実行してください。

---

**アシスタント:**

署名コマンドを実行しました：

```
> web-ext sign --channel unlisted ...

Building web extension from .
Waiting for validation...
Waiting for approval...
Signed xpi downloaded: web-ext-artifacts/aa268e0faa664fe99a12-0.2.6.xpi
```

**生成ファイル：** `web-ext-artifacts/aa268e0faa664fe99a12-0.2.6.xpi`

インストール方法：
1. Firefoxで `about:addons` を開く
2. 歯車アイコン（⚙）→「ファイルからアドオンをインストール」
3. 上記の `.xpi` ファイルを選択

また、チャットに記載されたAPIシークレットは漏洩リスクがあるため、APIキー管理ページで再生成することを推奨しました。

---

## ターン8

**ユーザー:**

ここまでのプロンプトと応答を docs/prompt.md に書き出してください。また、思考過程は docs/thinking.md に書き出してください。いずれも、APIシークレットなどの機微情報は、マスク化してください。

---

**アシスタント:**

本ファイル（`docs/prompt.md`）と `docs/thinking.md` を作成しました。APIキー・シークレットはマスク化済みです。
