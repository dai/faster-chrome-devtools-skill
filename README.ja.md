# faster-chrome-devtools-skill

Google が公式の [MCP サーバー](https://zeke.sikelianos.com/driving-chrome-with-an-agent/)を提供しており、AI エージェントからログイン済みの Chrome ブラウザをリモート操作できます。非常に便利ですが、デフォルトでは動作が遅くなりがちです。このスキルは、MCP を使ったブラウザ操作をより速く、より安全にするために作られました。

![HIC LIMAX NAVIGAT LENTE](images/chrome-snail-09.jpg)

## インストール

```sh
npx skills add zeke/faster-chrome-devtools-skill
```

## カバー内容

- [`take_snapshot`](https://developer.mozilla.org/en-US/docs/Glossary/Accessibility_tree)（平均 80ms）を [`take_screenshot`](https://pptr.dev/api/puppeteer.page.screenshot)（平均 1,118ms）より優先する — `take_snapshot` はページのアクセシビリティツリー（操作可能な要素のロール・名前・UID）を返します。`take_screenshot` はピクセル画像をレンダリングします。ページの状態確認や操作にはスナップショットを使い、見た目の確認が必要なときのみスクリーンショットを使ってください。

- スクリーンショットの安全性：PNG はロスレスでサイズが大きく、フルページの PNG は容易に 3〜7MB に達します。quality 75 の JPEG は通常 90%以上小さくなります。スクリーンショットが MCP の内部閾値（2MB）を超えるとディスクに保存されモデルには届かず、Claude の API 制限（5MB）を超えるとセッションが永続的に破壊されます。JPEG を使うべきタイミング、quality の設定方法、ファイルパスが返ってきたときの対処法をカバーしています。

- `navigate_page` には常にタイムアウトを設定する — 実際のセッションでタイムアウト未設定が 43 秒間ハングした事例があります。

- `list_pages` + `select_page` で既存のタブを再利用する（`new_page` は平均 3,500ms）。

- `wait_for` の内部動作（ポーリングではなく MutationObserver ベース）とコストが発生するケース。

- React コンポーネント・カスタムドロップダウン・合成イベントへの代替手段としての `evaluate_script`。

- サブ100msの正規インタラクションループ：`click` → `wait_for` → `fill` → `wait_for`。

- ブラウザセッションを使わず、`env.BROWSER.quickAction()` による [Browser Run Quick Actions](https://developers.cloudflare.com/browser-run/quick-actions/) を Worker 内で使うべきタイミング（単発のスクリーンショット・PDF・レンダリング済み HTML・Markdown・JSON 抽出・スクレイピング・リンク取得・スナップショット・クロール）。必要なブラウザバインディングと `2026-03-24` 以降の `compatibility_date` についてもカバーしています。

- [Cloudflare Browser Rendering](https://developers.cloudflare.com/browser-run/cdp/mcp-clients/) をリモートターゲットとして操作する：同じ `chrome-devtools-mcp` パッケージを CDP WebSocket でクラウド上のクリーンな匿名 Chromium に向けることができます。ローカル Chrome より好ましい場面と、知っておくべき注意点（`resize_page` は非対応・デフォルトビューポートが小さい・`navigator.clipboard` がハングする・ヘッドレス UA が検出される可能性あり）をカバーしています。

## 作成経緯

このスキルは、OpenCode のローカルセッション履歴（ツールコール・タイミング・レスポンスを記録した SQLite データベース）を分析することで作られました。

分析対象：

- 多数のセッションにわたる数百の `chrome-devtools_*` ツールコール
- 実際のコールタイムスタンプから計測したツールごとのタイミング分布
- ツールの遷移パターン（どのツールの後に何が呼ばれるか・そのレイテンシ）
- `wait_for` のタイムアウト失敗とその原因
- Claude の API 制限を超えるスクリーンショットによってセッションが永続的に破壊された実例
- `wait_for`・`navigate_page`・スクリーンショットのサイズゲーティングの内部動作を理解するための `chrome-devtools-mcp` のソースコード

## ライセンス

MIT
