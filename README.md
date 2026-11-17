# file-drop

Finder などからファイルをドロップすると、いま見ている space のフォルダへ**実際に転送**する Herdr Mod(受け口)。
リモート(ssh)の space なら、herdr のコア経由でリモートのフォルダへ送る。VS Code のエクスプローラへのドラッグ&ドロップに近い使い心地を目指す。

## 導入

    git clone git@github.com:naoto-anzai/file-drop.git ~/Projects/herdr-mods/file-drop
    sh ~/Projects/herdr-mods/mods-watcher/sync.sh ~/Projects/herdr-mods/file-drop ~/.config/herdr-app/mods/file-drop

## 状態

雛形のみ。設計は `TASKS.md`。
