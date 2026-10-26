# Teams CLI アクセスガイド

Microsoft Teams へのアクセスは Chrome DevTools Protocol (CDP) 経由で DOM を読み取る方式で行う。
Graph API にはチャット読み取りスコープがなく、Teams 新クライアントは Service Worker 経由のためネットワーク傍受も困難であり、DOM scraping が唯一の安定手法である。

## アーキテクチャ

```
teams-cli.py --[CDP/WebSocket]--> Chrome (VNC :2, port 9222) ---> teams.cloud.microsoft
```

## セットアップ

### 前提パッケージ

```bash
# Ubuntu/Debian
sudo apt install tigervnc-standalone-server google-chrome-stable
pip3 install --user --break-system-packages websockets
```

### 設定

1. `~/.config/agent-tools/config.json` に組織情報を記述（テンプレート: `config.example.json`）
2. `./setup.sh` でシンボリックリンクを作成

### ファイル配置

```bash
# dotfiles_public/agents/ からホームにシンボリックリンクを張る
# (./agents/setup_agents.sh を実行すると自動で作成される)
ln -sf ~/dotfiles_public/agents/teams-cli.py ~/teams-cli.py
ln -sf ~/dotfiles_public/agents/teams-start.sh ~/teams-start.sh
```

## 起動手順

```bash
# 1. VNC + Chrome を起動
~/teams-start.sh

# 2. 初回のみ: VNC で Teams にログイン
#    VNC 接続: localhost:5902
#    SSH トンネル: ssh -L 5902:localhost:5902 <host>

# 3. CLI で操作
python3 ~/teams-cli.py chats
```

## コマンド一覧

```bash
# 組織切り替え
python3 ~/teams-cli.py orgs                              # 利用可能な組織一覧
python3 ~/teams-cli.py org <key>                          # 組織を切替

# チャット
python3 ~/teams-cli.py chats                              # チャット一覧
python3 ~/teams-cli.py open "<name>"                      # チャットを名前で開く
python3 ~/teams-cli.py read                               # 現在開いているチャットを読む

# チーム・チャネル
python3 ~/teams-cli.py teams                              # チーム・チャネル一覧
python3 ~/teams-cli.py team <team> <channel>              # チームのチャネルを開いて読む
python3 ~/teams-cli.py team <team>                        # デフォルトで「一般」チャネル

# その他
python3 ~/teams-cli.py post "<body>"                          # 現在のチャネルに投稿
python3 ~/teams-cli.py post -s "<subject>" "<body>"           # 件名付きで投稿
echo "<body>" | python3 ~/teams-cli.py post -               # stdin から本文を読んで投稿
python3 ~/teams-cli.py post -s "<subject>" -                  # stdin + 件名
python3 ~/teams-cli.py thread "<query>"                       # スレッドを開いて返信を読む
python3 ~/teams-cli.py goto "<url>"                       # URL 直接指定
python3 ~/teams-cli.py dump                               # ページ全テキスト (debug)
```

組織のキーと名前は `~/.config/agent-tools/config.json` の `teams.orgs` で定義する。

### 送信前の確認 (誤送信防止)

Chrome は共有なので、前回の操作や人の操作で **組織 (テナント) や開いているチャットが変わっている** ことがある。外部テナントのチャットが開いたまま入力すると社外に送ってしまう。

- 送信の直前に `python3 ~/teams-cli.py org <key>` で目的の組織に戻す。切替直後はディープリンクが効かないことがあるので、切替が終わってからチャットを開き直す。
- 自前のスクリプトで入力する場合は、入力の直前に `document.title` (例: `チャット | <相手の名前> | Microsoft Teams`) に宛先名が含まれることを検証し、違えば中止する。
- 切替後に「このアプリを使用するためのアクセス許可がありません」等のダイアログが出ていたら、閉じてから操作する。
- bot (アプリ) との 1:1 チャットは `https://teams.microsoft.com/l/chat/0/0?users=28:<botAppId>` で開ける。

## MCP サーバー (Graph API 不要の外部テナント向け)

`teams-cli.py` と同じロジックを MCP (Model Context Protocol) サーバーとしても提供している。
Graph API のチャット読み取りスコープが無効な外部テナントでも、Teams 新クライアントの
DOM を CDP 経由で読む方式のため利用可能。Claude Code に限らず、stdio 対応の MCP クライアント
（Claude Desktop 等）から `tool_use` として直接呼び出せる。

実体: `~/dotfiles_public/agents/teams-mcp-server.py`（`~/teams-mcp-server.py` にシンボリックリンク）
依存: `pip3 install --user --break-system-packages mcp`（`setup_agents.sh` でインストール確認可）

### 起動前提

`teams-cli.py` と同じく、事前に `~/teams-start.sh` で VNC + Chrome を起動し、Teams にログイン済みであること。

### Claude Code への登録

```bash
claude mcp add teams -- python3 ~/teams-mcp-server.py
```

または `~/.claude.json` / プロジェクトの `.mcp.json` に直接:

```json
{
  "mcpServers": {
    "teams": {
      "command": "python3",
      "args": ["/home/shinjo/teams-mcp-server.py"]
    }
  }
}
```

### 提供ツール

`teams-cli.py` の各コマンドに対応する形で以下を公開している。

- `teams_orgs` / `teams_switch_org(org_key)`
- `teams_list_chats` / `teams_list_teams`
- `teams_open_channel(team_name, channel_name)` / `teams_open_chat(name)`
- `teams_read_current` / `teams_read_thread(query)` / `teams_goto(url)`
- `teams_post(body, subject)`
- `teams_copy_message_link(query)`
- `teams_reload` / `teams_dump_page`
- `teams_screenshot(out_path, full_page)` / `teams_save_images(out_dir)`

### 動作確認 (単体)

```bash
python3 ~/dotfiles_public/agents/teams-mcp-server.py
# stdio で待受する。手動テストは MCP クライアント経由 (ClientSession) で行う。
```

### 制約

`teams-cli.py` と同じ制約（VNC/Chrome 常駐必須、CDP ポート既定 9222 など）を継承する。
各ツール呼び出しごとに CDP 接続を張り直す実装のため、連続呼び出し時のレイテンシは
CLI を都度起動する場合と同程度。

## SharePoint ファイルダウンロード

Teams チャネルに添付されたファイルは SharePoint に保存されている。CDP 経由でブラウザの localStorage からトークンを取得し、SharePoint REST API でダウンロード可能。

### トークン取得

```python
# CDP evaluate で localStorage から MSAL キャッシュの .secret を取得
# key に対象ドメインと "sharepoint" を含むエントリ（-my. を除外）
```

### ファイルダウンロード

```python
SP_SITE = "https://<tenant>.sharepoint.com/sites/<site_id>"

# ファイルダウンロード
api_url = f"{SP_SITE}/_api/web/GetFileByServerRelativeUrl('{encoded_path}')/$value"
# Authorization: Bearer {token}

# フォルダ一覧
api_url = f"{SP_SITE}/_api/web/GetFolderByServerRelativeUrl('{encoded_folder}')/Files"
```

チャネルフォルダパス: `Shared Documents/{チャネル名}`

## 制約・既知の問題

- VNC + Chrome が起動している必要がある
- `open` は textContent の部分一致でクリック対象を選択（最短一致）
- Chromeセッションは`~/.config/agent-tools/chrome-teams`に保持する。ディレクトリには認証状態が含まれるため共有・commitしない
- Teams 用の CDP ポート既定値は `9222`

必要なら環境変数で上書きできる。

```bash
TEAMS_CDP_PORT=9324 TEAMS_VNC_DISPLAY=:6 TEAMS_VNC_PORT=5906 ~/teams-start.sh
TEAMS_CDP_URL=http://localhost:9324 python3 ~/teams-cli.py chats
```
- 大容量ファイル（100MB超）は base64 変換タイムアウトの可能性あり

## DOM 操作の勘所（新クライアント・英語 UI）

`teams-cli.py` のコマンドで足りないとき、CDP で直接 DOM を操作する際の知見である。UI の言語やバージョンで文言が変わるため、文言より `data-tid` を優先して使う。

### サイドバーとチャネル

- サイドバーは `[role="treeitem"]` の木構造で、`aria-level` が 1（セクション）/ 2（チャット・チーム）/ 3（チャネル）である。セクション見出しは英語 UI だと `Teams and channels`（c は小文字）になる。
- 折りたたまれたチームは `aria-expanded="false"` なので、先にクリックして展開してから level 3 のチャネルを探す。
- チャット一覧の `See more` は一度では全件展開されない。数回クリックする。1:1 チャットの表示名は相手のアカウント名（メールのローカル部など）になることがある。
- メッセージ本体は `[data-tid="message-pane-body"] [data-tid="message-pane-list-viewport"]` 内にある。仮想スクロールで描画範囲外の投稿は DOM から消えるため、全履歴を取るときは `scrollTop` を少しずつ動かして重複を除きながら集める。

### スレッド

- 返信のある投稿には `N replies` ボタン（id `rc-sum-cont-btn-<ミリ秒タイムスタンプ>`）がある。クリックすると右側に `[data-tid="right-rail-message-pane-body"]` としてスレッドが開く。id のタイムスタンプは投稿時刻なので、並べ替えや追跡のキーに使える。
- 返信のない投稿にスレッド返信するには、`Escape` を送り、マウスを一度別の場所へ動かしてから投稿の中央へ `mouseMoved` でホバーする。表示された `[data-tid="message-actions-threaded-reply"]` を押す。
- 右パネルが開いたままだと、ホバーした別の投稿ではなく、開いているスレッドに書き込んでしまうことがある。開く前に `[data-tid="thread-list-pane-toggle-button"]` で閉じ、開いた後はスレッド先頭のテキストを照合してから入力する。
- 返信欄は右パネル内の `[contenteditable="true"]` である。`Input.insertText` で入力し、改行は Shift+Enter を1回（2回だと空行が増える）、送信は Enter で行う。送信後にスレッド末尾を読み直して反映を確認する。

### 外部への投稿の注意

- 社外のゲストがいるチャネルへの投稿では、内部のファイル名・CI の実行 ID などの詳細や、人物の写った画像は控える。エージェントの権限分類器にも止められやすい。
- 一度に大量の操作をすると時間がかかる（スレッド200件の取得で約30分）。新着の確認だけなら Graph API（teams-graph-access.md）の方が速い。

### 書式付きの下書きを返信欄に入れる（送信しない）

- ユーザーに確認してから送りたいときは、返信欄に下書きだけ入れて止める。投稿自体は Graph API の方が確実なので、下書きが要るときだけこの方法を使う。
- リンクや箇条書きを保ったまま入れるには、返信欄に `paste` イベントを合成して `text/html` を渡す。Teams のエディタがそのまま取り込む。

  ```js
  const dt = new DataTransfer();
  dt.setData('text/html', html); dt.setData('text/plain', 'draft');
  editor.focus();
  editor.dispatchEvent(new ClipboardEvent('paste', {clipboardData: dt, bubbles: true, cancelable: true}));
  ```

- `document.execCommand('insertHTML')` や `innerHTML` の代入は TrustedHTML の制約で例外になる。`navigator.clipboard.write` は CDP からだと「Document is not focused」で失敗する。
- SharePoint のリンクはファイルカードに自動展開される。社外チームのファイルだと「アクセス権を付与する」が出るので、スクリーンショットで確認してユーザーに伝える。
