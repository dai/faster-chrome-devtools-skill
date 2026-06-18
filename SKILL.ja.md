---
name: faster-chrome-devtools-skill
description: >
  Chrome DevTools MCP のパフォーマンス・安全性ガイド。chrome-devtools_* ツール（take_snapshot、
  take_screenshot、navigate_page、wait_for、click、fill、new_page、list_pages、
  select_page、evaluate_script）を使用する前に必ずこのスキルをロードしてください。
  ツールのプレフィックスはMCPサーバーの設定キーによって異なる場合があります。
  セッションを永続的に破壊するスクリーンショットサイズ制限、ナビゲーションタイムアウトの落とし穴、
  ブラウザ接続失敗時のリカバリー方法、よくあるブラウザ自動化タスクの最速パターン、
  Workers からの Browser Run Quick Actions、リモート Cloudflare Browser Rendering
  ターゲットの操作上の注意点を網羅しています。
---

# Chrome DevTools MCP: 高速パターン集

## ツール名について

ツール名のプレフィックスは MCP サーバーの設定キーによって異なります。デフォルトの単一インスタンス `chrome-devtools-mcp` では `chrome-devtools_*`（例：`chrome-devtools_take_snapshot`）になります。リモートの Cloudflare Browser Rendering ターゲットを指す第2インスタンスを追加した場合は、割り当てたキー名がプレフィックスになります。このガイドでは `take_snapshot`、`navigate_page` などプレフィックスなしのツール名で記載しています。実際に使用するプレフィックスに読み替えてください。

## ツール速度リファレンス

実際のセッションデータから計測した中央値（デフォルトのビューポートサイズ時）：

| ツール                  | 中央値   | 備考                                             |
| ----------------------- | -------- | ------------------------------------------------ |
| `take_snapshot`         | 34ms     | 最速のページ確認手段。スクリーンショットより優先。 |
| `list_console_messages` | 73ms     | 軽量                                             |
| `wait_for`              | 97ms     | テキストが既に存在する場合は即時解決             |
| `fill`                  | 245ms    |                                                  |
| `evaluate_script`       | 301ms    | 高速。React コンポーネントへの代替手段として活用 |
| `click`                 | 304ms    |                                                  |
| `take_screenshot`       | 722ms    | 低速。見た目の確認が必要な場合のみ使用           |
| `navigate_page`         | 1,219ms  | 変動が大きい。必ず `timeout` を設定すること      |
| `new_page`              | 2,380ms  | コストが高い。可能な限り既存タブを再利用すること |
| `list_pages`            | 3,432ms  | よく使うツールの中で最も遅い。ループ内での使用は避けること |

## スクリーンショットの安全性

PNG はロスレスで非圧縮のため、1280px 幅の一般的なページのフルページ PNG は容易に 3〜7MB に達します。JPEG と WebP はロッシー圧縮を使用しており、quality 75 の JPEG は同等の PNG と比べて通常 90%以上小さくなります。ページ確認用途では画質の劣化はほぼ感じられません。

これはパイプラインの2つのサイズ閾値に関わるため重要です：

- **2MB（MCP 閾値）**：スクリーンショットが 2MB 以上になると一時ファイルに保存され、モデルにはファイルパスのみが返されます。画像はモデルに届きません。警告なしにサイレントで発生します。
- **5MB（Claude API 制限）**：インラインのスクリーンショットが base64 で 5MB を超えると、API がリクエスト全体を拒否し、セッションが永続的に回復不能になります。コンパクションしても同じ画像が再生されるため解決しません。

モデルに画像を表示する場合は、必ず JPEG または WebP に quality を指定して使用してください：

```
// 安全
take_screenshot({ format: "jpeg", quality: 75 })

// 危険 — PNG は非圧縮、fullPage はさらにサイズを増大させる
take_screenshot({ fullPage: true })
```

`fullPage: true` は本当に全ページが必要なときだけ使い、必ず `format: "jpeg", quality: 75` とセットで指定してください。

ファイルパスが返ってきた場合は、スクリーンショットが 2MB を超えたことを意味します。quality 60 の JPEG でリトライしてください。

## スナップショット優先

`take_snapshot` はページの[アクセシビリティツリー](https://developer.mozilla.org/en-US/docs/Glossary/Accessibility_tree)（他のツールに渡せる要素のロール・名前・UID）を返します。`take_screenshot` は [Puppeteer 経由でピクセル画像をレンダリング](https://pptr.dev/api/puppeteer.page.screenshot)します。

ページの状態を把握したい場合は `take_snapshot` を使用してください。`take_screenshot` は視覚的な見た目（画像・CSS レンダリング・canvas）が重要なときのみ使用してください。

```
// ページ状態の確認 — 高速
take_snapshot()

// グラフが正しくレンダリングされているか確認 — スクリーンショットが適切
take_screenshot({ format: "jpeg", quality: 75 })
```

`navigate_page` の後、`take_snapshot` は約 15ms で解決します。同じ情報を得るのに `wait_for` + `take_screenshot` の組み合わせでは平均 3,800ms かかります。

## navigate_page には必ずタイムアウトを設定する

タイムアウト未設定の場合、`navigate_page` は無期限にブロックする可能性があります。タイムアウト未設定のローカルサーバーへのナビゲーションが 43 秒間ハングした事例があります。

```
// 常にタイムアウトを含める
navigate_page({ type: "url", url: "https://example.com", timeout: 15000 })

// 危険 — タイムアウトなし
navigate_page({ type: "url", url: "http://localhost:3000" })
```

状況別の推奨タイムアウト：

| 状況                          | タイムアウト |
| ----------------------------- | ------------ |
| ローカル開発サーバー          | 10,000ms     |
| 通常のウェブページ            | 15,000ms     |
| 重いページ・リソースが多いページ | 30,000ms   |
| OAuth / 外部リダイレクトフロー | 60,000ms    |

## タブの再利用

`new_page` の中央値は約 2,400ms、`list_pages` はさらに遅いです。対象の URL が既に開いているタブがある場合はそれを使用してください。

```
// まず確認する
list_pages()
select_page({ pageId: <id> })

// URL が既に開いていない場合のみ新しいタブを開く
new_page({ url: "https://example.com" })
```

## ブラウザに接続できない場合

ローカルサーバーで最も多い障害は、`list_pages`、`navigate_page`、`new_page` での `Not connected` エラーや `MCP error -32001: Request timed out` の繰り返しです。MCP サーバーが Chrome インスタンスに到達できていないことを意味します。リモートデバッグが有効になっていないか、MCP にブラウザアクセスが許可されていない可能性があります。

リトライしても解決しません。1回リトライしたら止めてリカバリーしてください：

1. ユーザーにリモートデバッグを有効にするよう依頼します（Chrome の場合は `chrome://inspect/#remote-debugging`）。MCP のブラウザアクセス許可を求めるプロンプトが表示された場合は許可してもらい、その後1回リトライします。
2. クリーンまたは匿名のブラウザで問題ない場合は、リモートの Cloudflare Browser Rendering インスタンスが設定されていればそちらにフォールバックします。

同じエラーでツールコールを繰り返し消費しないでください。ほぼ常に、再試行ではなくユーザーによるアクセス許可の操作が必要です。

## wait_for の仕組み

`wait_for` はポーリングループではなく MutationObserver ベースです。DOM に一致するテキストが現れた瞬間に解決します。対象のコンテンツが既に存在しているか、すぐに現れる場合は 40〜100ms で解決します。

コストが発生するのは、期待したコンテンツが現れずタイムアウトが経過した場合のみです。操作が実際にかかりうる時間を反映したタイムアウトを設定してください：

```
// UIアップデートをトリガーするクリックの後 — 短いタイムアウトで十分
wait_for({ text: ["Success", "Done"], timeout: 5000 })

// 低速なAPIにヒットするフォーム送信の後
wait_for({ text: ["Order confirmed"], timeout: 15000 })

// OAuthフローの開始後 — 外部リダイレクトの時間が必要
wait_for({ text: ["refresh_token"], timeout: 60000 })
```

アクセシビリティツリーに現れないもの（バックグラウンドプロセス、DNS 伝播、外部サービスの完了）には `wait_for` を使わないでください。代わりに `evaluate_script` で JS の条件をポーリングしてください。

## evaluate_script によるハードケースへの対処

React のカスタムコンポーネント、ヘッドレスドロップダウン、合成イベント入力にはアクセシビリティツリーが不十分な場合があります。そのような場合の代替手段として `evaluate_script` を使用してください。

プログラムによるクリック（React-select などの場合）：

```js
evaluate_script({
  function: () => {
    const option = document.querySelector('[class*="option"]');
    option?.click();
  }
})
```

React の合成イベントを使ったレンジスライダー：

```js
evaluate_script({
  function: () => {
    const input = document.querySelector('input[type="range"]');
    const setter = Object.getOwnPropertyDescriptor(HTMLInputElement.prototype, 'value').set;
    setter.call(input, '75');
    input.dispatchEvent(new Event('input', { bubbles: true }));
  }
})
```

a11y ツリーにない状態を読み取る：

```js
evaluate_script({
  function: () => document.querySelector('.status')?.dataset.state
})
```

## 正規インタラクションパターン

自動化セッションで観察されたサブ100msループ — 次のアクションの前に必ず `wait_for` で状態遷移を確認し、任意のスリープは使わない：

```
click({ uid: "..." })                                     // ~105ms
wait_for({ text: ["Enter symbol"], timeout: 3000 })       // ~60ms
fill({ uid: "...", value: "AAPL" })                       // ~105ms
press_key({ key: "Enter" })                               // ~105ms
wait_for({ text: ["Sell All", "Action"], timeout: 3000 }) // ~65ms
fill({ uid: "...", value: "Sell" })                       // ~105ms
click({ uid: "..." })                                     // ~105ms
wait_for({ text: ["Order confirmed"], timeout: 5000 })    // ~55ms
```

## アンチパターン

| アンチパターン                              | 代わりに使うもの                                                   |
| ------------------------------------------- | ------------------------------------------------------------------ |
| DOM 状態確認に `take_screenshot()` を使う  | `take_snapshot()`                                                  |
| `take_screenshot({ fullPage: true })`       | `take_screenshot({ fullPage: true, format: "jpeg", quality: 75 })` |
| タイムアウトなしの `navigate_page({ url })` | 必ず `timeout` を含める                                            |
| タブが既に開いているのに `new_page()` を使う | `list_pages()` してから `select_page()`                           |
| 非同期の外部イベント待ちに長い `wait_for` を使う | `evaluate_script` で JS 条件をポーリング                      |
| a11y 経由で React コンポーネントをクリック | `evaluate_script` で直接 DOM 操作                                  |
| Worker から Browser Run REST API を `fetch()` で呼ぶ | バインディングで `env.BROWSER.quickAction(...)` を使う  |
| 単発のスクレイピング/PDF 作業に CDP セッションを使う | Browser Run Quick Actions を使う                           |

## Workers からの Browser Run Quick Actions

URL や HTML 文字列からステートレスなブラウザ出力を得るタスクで、かつ Cloudflare Worker を書いている場合は、Chrome DevTools MCP・CDP・Puppeteer・Playwright・REST API 呼び出しではなく、ブラウザバインディング経由の Browser Run Quick Actions を使用してください。

Workers は `env.BROWSER.quickAction(action, options)` で Quick Actions を直接呼び出せます。API トークンが不要で、外部 HTTP リクエストも発生せず、Cloudflare のネットワーク上で完結します。

Quick Actions の用途：

- スクリーンショット：`"screenshot"`
- PDF：`"pdf"`
- レンダリング済み HTML：`"content"`
- Markdown 抽出：`"markdown"`
- AI 構造化 JSON 抽出：`"json"`
- CSS セレクタースクレイピング：`"scrape"`
- リンク抽出：`"links"`
- HTML とスクリーンショットの同時取得：`"snapshot"`
- クロール：`"crawl"`

必要な Worker の設定：

```jsonc
{
  "compatibility_date": "2026-03-24",
  "browser": {
    "binding": "BROWSER"
  }
}
```

使用例：

```js
const screenshot = await env.BROWSER.quickAction("screenshot", {
  url: "https://www.cloudflare.com/",
});
```

判断基準：

- レンダリングされたページから1つの出力が必要：`quickAction()` を使う。
- ログイン状態、複数ステップのクリック、DOM ミューテーション、アクティブなタブのデバッグ、エージェントによる視覚確認が必要：Chrome DevTools MCP またはブラウザセッションを使う。
- Quick Action で表現できない Worker 内のブラウザ自動化が必要：Puppeteer・Playwright・CDP を使った Browser Run を使う。
- Worker から Quick Actions を呼ぶのに `fetch()` で Browser Run REST API を叩かないこと。バインディングを使うこと。

## Cloudflare Browser Rendering（リモートターゲット）

`chrome-devtools-mcp` の第2インスタンスを、ローカルの Chrome ではなく [Cloudflare Browser Rendering](https://developers.cloudflare.com/browser-run/cdp/mcp-clients/) 上で動作する Chromium に CDP WebSocket で向けることができます。同じパッケージ・同じツール群で、異なるランタイムです。ツールプレフィックスは割り当てた設定キーによって決まります。上記のパターンはすべて適用されますが、以下の注意点があります。

**リモート（Cloudflare）を優先すべき場合：**

- ローカルの Chrome がユーザーの実際のセッションで使用中で、邪魔したくない場合。
- ログイン済み Cookie を持たないクリーンな匿名ブラウザが必要な場合（ローカルの autoConnect サーバーはユーザーのセッションを引き継ぐため）。
- CI 上、サーバー上、またはローカル Chrome がない環境で動作している場合。
- ユーザーの実際のブラウザに影響を与えずに geolocation・userAgent・viewport のエミュレーションが必要な場合。

**ローカルを優先すべき場合：**

- ユーザーの既存のログイン済みセッション（銀行、社内ツール、有料サイト）にアクセスが必要な場合。
- ユーザーが今まさに見ているものをデバッグしている場合。
- レイテンシが重要な場合 — ローカルの CDP ラウンドトリップはリモートより高速です。

**リモートターゲットを操作する上での既知の注意点：**

- `resize_page` は `Browser.setContentsSize wasn't found` で失敗します。ステータスは `completed` と表示されますがエラーは出力に埋め込まれており、ステータスの成功表示は誤解を招きます。ステータスを信用せず出力を確認してください。代わりに `emulate({ viewport: "1280x800x1" })` を使用してください。以降のすべてのツールレスポンスにエミュレートされたビューポートがエコーされますが、ノイジーなだけで実害はありません。
- デフォルトのビューポートは小さいです（780x493）。セッション開始時に早めに `emulate` で実際のビューポートを設定してください。
- `evaluate_script` 内での `navigator.clipboard.readText()` はハングし、`MCP error -32001: Request timed out` を返します（パーミッションプロンプトの UI に到達できません）。セッション自体は次のコールで回復しますが、タイムアウト分を消費します。ソース要素を直接確認するなど別の方法でクリップボードの状態を読むか、確認自体をスキップしてください。
- リモートブラウザは `X11; Linux x86_64` 上の `HeadlessChrome/126` として識別されます。UA やヘッドレス Chrome に対して異なる動作をするサイトは、ローカルとは異なる挙動を示します。
- `emulate({ geolocation: "lat x lon" })` は動作しますが、レスポンスにはエコーされません。実際に位置情報が必要な場合は `evaluate_script` から `navigator.geolocation.getCurrentPosition` で確認してください。
- `lighthouse_audit` は動作し、完全な HTML + JSON レポートを生成します。独立した Lighthouse をセットアップせずにアドホックな監査を行うのに便利です。

**セットアップメモ**（エージェントが自発的に行う作業ではなく、参考情報）：

- 設定は MCP クライアント（OpenCode、Claude Desktop、Cursor 等）側に `chrome-devtools-mcp` エントリとして記述します。`--wsEndpoint=wss://api.cloudflare.com/client/v4/accounts/<ACCOUNT_ID>/browser-rendering/devtools/browser?keep_alive=600000` と、`Browser Rendering - Edit` API トークンを持つ `--wsHeaders` を指定します。
- `keep_alive` のデフォルトは 600000ms（10分）のアイドル後にセッションがリサイクルされます。長時間の自動化ではこの値を増やしてください。

**新規リモートセッションのサニティチェックパターン：**

```
list_pages()                                    // about:blank から始まることを確認
emulate({ viewport: "1280x800x1" })             // resize_page は動作しない
navigate_page({ url, timeout: 15000 })          // 標準タイムアウト
take_snapshot()                                 // ロード確認
evaluate_script({ function: () => ({
  ua: navigator.userAgent,
  viewport: { w: innerWidth, h: innerHeight }
})})                                            // ヘッドレスの識別情報を確認
```
