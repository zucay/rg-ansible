# rg-ansible

ARM組み込みLinux（ssh/systemd/python導入済み、頻繁に初期化される）向けの構成管理雛形。
初期化のたびに `ansible-playbook` を再実行するだけで、目的の状態に戻せることを目指しています。

## 対象機
- IP: `192.168.0.40`
- SSH: port `22`
- 初期状態のアカウント: `root` / `root`
- OS: Ubuntu 22.04 LTS (jammy) / ARM

## 現在やっていること
1. `apt_sources` ロール: `/etc/apt/sources.list` を国内ミラー（ftp.yz.yamagata-u.ac.jp のubuntu-ports）+ 公式ミラー（ports.ubuntu.com）に差し替え
   - ARM版UbuntuはアーキがARMのため`archive.ubuntu.com`ではなく`ports.ubuntu.com`（ARM用の別系統アーカイブ）を使う点に注意
2. `locale` ロール: デフォルト言語を英語（元は中国語想定）に変更。`en_US.UTF-8` は生成済み前提（SDカードが遅い機体があり、`locale-gen` を毎回走らせるコストを避けるため）
3. `syncthing` ロール: 公式apt リポジトリ（apt.syncthing.net）を追加して最新の`syncthing`を導入し、`syncthing@root.service` として常駐化。jammyのuniverseにあるバージョン（1.18.0）は設定生成に使う`generate`サブコマンド未対応のため公式リポジトリが必要。GUIは家庭内LANからアクセスできるよう `0.0.0.0:8384` で待ち受け、Basic認証を `root` / `rootroot`（`inventory/hosts.ini` のSSH認証情報と同じ）に設定。LAN外に公開する場合は認証情報を必ず変更してください
   - 共有フォルダ（`syncthing_folders` 変数、デフォルトで `RetroArch Saves` / `RetroArch States` を `/mnt/sdcard/` 配下に設定）も `syncthing cli` 経由で宣言的に追加。GUIでの手動フォルダ追加が不要になる
   - リモートデバイスとの共有（デバイスIDのペアリング・承認）はロールの対象外。新しい機器を追加するときはGUIから1回だけ行う

## 前提
- 制御ノード（このリポジトリを実行するPC）に `sshpass` が必要（パスワード認証のため）
  ```
  sudo apt install sshpass   # Debian/Ubuntu
  ```

## 実行方法
```
ansible-playbook playbook.yml
```

疎通確認のみ:
```
ansible embedded -m ping
```

## 設計メモ
- `ansible.cfg` で `host_key_checking = False` にしています。初期化のたびにSSHホスト鍵が変わるため、これがないと2回目以降の実行が `REMOTE HOST IDENTIFICATION HAS CHANGED` で失敗します。
- `inventory/hosts.ini` に `root/root` を平文で置いています。これは工場出荷時の既知パスワードなので現時点では実害は小さいですが、実機のパスワードを変更した場合は `ansible-vault` に移行してください。
- 各ロールは再実行しても安全（冪等）です。初期化後に同じコマンドをもう一度叩けば元の状態に戻ります。
- タスクは `apt_sources` / `locale` / `syncthing` タグで個別実行もできます（例: `--tags syncthing`）。
- `syncthing` ロールは初回のみ `syncthing generate` で設定ファイル（GUI認証情報を含む）を生成し、以降は既存の `config.xml` を上書きしません。`syncthing_folders` に無いフォルダやGUIで追加したデバイス設定は再実行しても消えません（追加のみで削除はしない設計）。

## 今後の拡張候補（未実装）
- SSH鍵配布・パスワード認証の無効化
- root以外の作業ユーザー作成
- 追加パッケージのインストール、systemdサービスの配置
- `ansible-vault` によるパスワード管理
