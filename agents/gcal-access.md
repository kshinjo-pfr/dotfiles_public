# Google Calendar アクセスガイド

Google Calendar へのアクセスは [gcalcli](https://github.com/insanum/gcalcli) で行う。

## インストール

```bash
pip3 install --user gcalcli[vobject]
```

## セットアップ

### 1. Google Cloud で OAuth2 クライアントを作成

1. [Google Cloud Console](https://console.developers.google.com/) でプロジェクトを作成（Gmail と共用可）
2. Google Calendar API を有効化
3. OAuth 同意画面を設定（テストユーザーに自分を追加）
4. OAuth クライアント ID を作成（種類: デスクトップアプリ）
5. Client ID と Client Secret を控える

### 2. 初期認証

```bash
gcalcli --client-id=<client_id>.apps.googleusercontent.com init
```

Client Secret の入力を求められた後、ブラウザで Google アカウントにログインする。

トークンは `~/.local/share/gcalcli/oauth` に保存される。

## コマンド一覧

```bash
# カレンダー一覧
gcalcli list

# 予定表示
gcalcli agenda                                      # 今日以降の予定
gcalcli agenda "2026-03-08" "2026-03-15"             # 期間指定
gcalcli calw                                         # 週表示
gcalcli calm                                         # 月表示

# 予定の追加
gcalcli quick "Meeting tomorrow 3pm"                 # 自然言語で追加
gcalcli add                                          # 対話式で追加

# 予定の編集・削除
gcalcli edit "event title"
gcalcli delete "event title"

# ICS インポート
gcalcli import file.ics

# 特定のカレンダーを指定
gcalcli --calendar="Work" agenda
```

エージェントから使う場合は `--nocolor --tsv` を付けるとパースしやすい。

## Claude の Google Calendar コネクタを使う場合

- 他人のカレンダーも `calendarId` にメールアドレスを渡せば読める（閲覧権限がある場合）。複数人の空き時間を突き合わせるときは、各人の `list_events` を同じ期間で取る。
- 1 週間分でも出力が大きく、ファイルに退避されることが多い。JSON を python で読み、`start.dateTime`・`summary`・`location`・自分の `responseStatus` だけ抜き出す。`transparency: transparent` の予定は空き扱いでよい。
- 出社場所は `eventType: ["WORKING_LOCATION"]` を指定して取る。未登録の人もいるので、会議の `location`（会議室名）からも推測し、確定できないときはそう伝える。
- 会議室の空きは、会議室のリソースカレンダー ID（例: `c_...@resource.calendar.google.com`）で `list_events` すれば分かる。ID は既存予定の `attendees` に含まれている。
