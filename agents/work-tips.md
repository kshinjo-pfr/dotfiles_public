# 作業の勘所 (プロジェクトに依らないもの)

エージェント (Claude Code・Codex) と人が、どのプロジェクトでも踏みやすい落とし穴。プロジェクト固有のものは
そのリポジトリの文書 (`AGENTS.md`・`docs/` など) に置く。パスワード・トークンはここに書かない。

## 1. シェルとファイルの編集

- **ヒアドキュメントの中にヒアドキュメントを入れない。** 区切り (`EOF`) がぶつかって途中で切れる。長いスクリプトは
  一時ディレクトリにファイルとして書いてから実行する
- **`sed` の区切り文字が置き換える文字列に入っていると壊れる** (`s|a|b|` で正規表現に `|` の選択がある、など)。
  複数行や記号の多い置き換えは、Python で「元の文字列がちょうど 1 回あること」を確かめてから置き換える:
  ```python
  s = open(p).read(); assert s.count(old) == 1, old; open(p, 'w').write(s.replace(old, new))
  ```
- **YAML・JSON を grep で数えない。** 書き方 (インデント・改行・クォート) で外れて「0 件」になる。`yq`・`python3 -c 'import yaml'`
  で読むか、`grep -o` で候補を全部出して目で見る
- **終わらないプロセスを待つスクリプトには必ず時間の上限を付ける** (`timeout`、`wait` の前に kill)。止まらない子プロセスで全体が固まる

## 2. sudo とパスワード

- **`echo <pw> | sudo -S tee <file>` はしない。** パスワードと書き込む内容が同じ標準入力を取り合い、空のファイルができる。
  内容は手元のファイルに書いてから `sudo install -m644 <src> <dst>`、または `sudo bash -c '... > <file>'`
- **パスワードが通らなかったら、ほかの候補を試さない。** 続けて失敗するとロックされることがある。人に聞く
- パスワードをコマンドの引数や画面に出さない (`ps` や履歴に残る)

## 3. git・GitHub

- **worktree は submodule が空で始まる。** ビルドの前に `git submodule update --init <要るもの>`
- 作業は worktree か別ブランチで。手元の main のチェックアウトを汚さない (他の作業や人がそれを前提にしている)
- リポジトリのマージ方式 (squash・rebase のみ、など) と CI の lint (clang-format など) を、push の前に確かめる
- PR のマージ・push・公開リポジトリへの送り出しは、頼まれたときだけ。公開リポジトリに社内の変更・名前を入れない

## 4. リモートの機器 (SSH・ネットワーク)

- **Tailscale・mDNS で届かないときは、まず相手の電源とネットワーク。** 手元で直せることは少ない。DHCP の機器は IP が変わる
  ので、IP を決め打ちしない (`<ホスト名>.local` か Tailscale の名前)
- **Wi-Fi の省電力で、リモートの PC がたびたび切れる。** NetworkManager の `wifi.powersave = 2` (切る)
- **SSH の鍵は `-i` で明示する。** 既定の鍵が使われて弾かれた、を「鍵が登録されていない」と取り違えやすい。ポートノックが要る相手もある
- 社内のネットワークを総当たりで調べない (スキャンは止める)。相手のアドレスは人に聞く
- 構成管理 (Ansible など) は対象を 1 台に絞って流す (`-l <host>`)。付け忘れると全台に当たる
- 動いている機器のサービスの入れ替え・再起動は、使っている人の作業 (記録など) を止める。する前に確かめる

## 5. Docker とビルド

- **メモリの少ない機械でのビルドは並列を絞る** (`-j4`、`--build-arg BUILD_JOBS=...`、colcon なら `--parallel-workers`)。
  C++ の大きなライブラリは 1 ジョブ 1 GB 前後食う。`free -h` を見ながら、ほかの重い処理と同時にしない
- `.env` など機械にだけ置くファイルを `rsync --delete` で消さない (`--exclude` を付ける)
- 作ったイメージで動作を確かめるまで、前のイメージを別のタグで残しておく (すぐ戻せる)

## 6. ROS 2

- **`ros2 topic echo` で何も出ないときは QoS を疑う。** 送り手が Best Effort なら `--qos-reliability best_effort`
- **rmw_zenoh**: ルーター (`zenohd`) が動いていないと何も見えない。起動直後はディスカバリに数秒かかる。ノードが SIGINT で
  止まらないことがあるので、SIGINT → 数秒 → SIGTERM → SIGKILL
- **rosbag2 の `metadata.yaml` が無い** (記録中に落ちた) ときは `ros2 bag reindex <bag> -s mcap`。データは残っている
- 圧縮の preset `zstd_small` は CPU を食い、遅い PC では記録が詰まる。`zstd_fast`
- `/robot_description` などの latched なトピックは bag の頭にしか無い。`--start-offset` で途中から再生すると出ない
- 帯域の細い所 (Wi-Fi・VPN) から画像・点群を購読しない。送り手の記録まで欠ける。`ros2 topic hz` も購読になる
- `camera_calibration_parsers` はファイルの拡張子で形式を決める (`.yaml` 以外は INI)。テストの一時ファイルは
  `mkstemps` などで拡張子を付ける

## 7. USB・センサーの機器

- USB3 の機器が遅い・落ちるときは、USB2 でつながっていないか (`lsusb -t` の速度)。ケーブルと差し込みを疑う
- Ethernet の機器のリンク速度 (`ethtool <if>`) が 100 Mb/s なら、ケーブル (Gigabit は 4 対) を疑う
- USB ハブの給電を入れ直す (`uhubctl -a cycle`) と、見えなくなった機器が戻ることがある
