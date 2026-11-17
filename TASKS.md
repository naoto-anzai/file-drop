# file-drop の設計と、やること

決まりごと: 親の `~/Projects/herdr-mods/CLAUDE.md`(Story 契約 v2: モックと `storybook()` は `main.js` 内、`./mock.js` は作らない。
同梱 Mod / layout は直接編集せず `mod.clone` / `layout.clone` で複製してから触る)。

## 目的(第 1 段)

Mac のファイルをドロップすると、**見ている space のフォルダ**へ転送する受け口 Mod。
- space がリモート(`spaces()[i].machine` が非 null)なら、そのマシンのフォルダへ。ローカルならコピー。
- 方向は **Mac → リモートのみ**。リモート → Mac は今回やらない。
- 範囲は**独立した受け口 Mod**。Files(`files` / `files-md`)の行へ直接ドロップする案は、あとの段で別に検討する。

## 実機で分かっていること(2026-11-17)

- **ドロップの受け取り**: 平らな要素に `.on_drop((event, cx) => event.paths)`(`gpui-kit.d.ts`)。Finder から落としたファイルの絶対パスが来る。
  同梱の `files` / `files-md` には、ドロップの処理が無い。
- **ssh は使えない**: Mac から `ssh naoto-wsl` は鍵認証が通らない(パスワード認証。machine-login Mod がそのために存在する)。`scp` / `rsync` は無人では使えない。
- **転送は herdr のコア経由**: `hc file write --on <マシン> <パス> < ファイル`(stdin をそのままファイルにする)。
  - バイナリで壊れない(200KB のランダムバイナリで SHA-256 一致を確認)。
  - **1 回 8 MB まで**(超えると `at most 8 MB piped in`)。追記や offset は無い。
  - 親フォルダは自動で作る。
- **大きいファイル**: 約 7 MB ずつに分割して、リモートの一時フォルダへ送り、`hc exec --on <マシン> sh -c 'cat part* > 本体 && rm part*'` で連結する。最後に `sha256sum` で照合する。
- **フォルダのドロップ**: `tar` で 1 本にまとめて(8 MB 超なら上の分割で)送り、リモートで `tar -xf`。ファイルを 1 つずつ送ると往復が多くて遅い。
- **Mod の `exec` には 30 秒の制限と stdin が無い**(`herdr.d.ts`)。stdin を渡すなら `exec("sh", ["-c", "hc file write --on M PATH < FILE"])`。
  **長い転送は `hc run --bg` の端末で動かす**(進捗をファイル `~/.herdr/xfer/<id>.json` に書き、Mod がポーリングして表示する)。
- `hc exec --on` の簡易シェルには `~/.local/bin` が入っていない。リモートのコマンドはフルパスで呼ぶ。
- リモートのフォルダの `rm -r` は、このセッションの権限では拒否される。一時ファイルは個別に `rm` して `rmdir` する。

## 作るもの

1. `props` で受けられる Mod: `props.folder` と `props.machine`(省略時は、見ている space の folder と machine)。
2. 受け口の見た目: 破線の枠に「ここにドロップ」、送り先(`naoto-wsl:/home/naoto/projects/x`)、直近の転送の一覧(ファイル名・サイズ・進捗・成功/失敗)。
3. 転送の実体: 小さなスクリプト(Python 1 ファイルが楽。Mod 内に置けないなら `hc run --bg` で起動する別ファイルでよいが、
   mods-watcher のファイル単位 symlink では、別ファイルの `import` が `herdr-app check` に弾かれることに注意。**スクリプトは Mod のフォルダの外(例: リポジトリの `bin/`)に置き、絶対パスで呼ぶ**)。
4. 上書きの確認: 同名のファイルがあれば、上書き / 別名 / 中止を聞く。
5. 失敗の扱い: マシンに届かない、容量不足、権限、途中で切れた、を分けて表示。照合(SHA-256)が合わなければ失敗にする。
6. `storybook.json` に状態(転送なし/進行中/成功/失敗/上書き確認)を列挙し、モックを `main.js` 内に作る。

## 進め方

- 変更後は `herdr-app check` と `herdr-app shot`(自分の space を選んでから)で確認する。
- 実機の転送テストは、リモートの `/tmp` 配下に**使い捨てのファイル**で行い、終わったら個別に消す。実在のプロジェクトのフォルダには送らない。
- 画面の挙動が変わる操作(layout 切替、レイアウトへの常駐など)は、実行前に報告する。
