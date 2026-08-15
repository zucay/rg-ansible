# rg-ansible

ARM組み込みLinux（ssh/systemd/python導入済み、頻繁に初期化される）向けの構成管理雛形。
初期化のたびに `ansible-playbook` を再実行するだけで、目的の状態に戻せることを目指しています。

## 対象機
- IP: `192.168.0.40`
- SSH: port `22`
- 初期状態のアカウント: `root` / `root`
- OS: Ubuntu 22.04 LTS (jammy) / ARM

## 現在やっていること
1. `apt_sources` ロール: `/etc/apt/sources.list` を国内ミラー（ftp.jaist.ac.jp のubuntu-ports）+ 公式ミラー（ports.ubuntu.com）に差し替え
   - ARM版UbuntuはアーキがARMのため`archive.ubuntu.com`ではなく`ports.ubuntu.com`（ARM用の別系統アーカイブ）を使う点に注意
2. `locale` ロール: `en_US.UTF-8` と `ja_JP.UTF-8` を生成し、デフォルト言語を英語（元は中国語想定）に変更。日本語表示自体はできるようにしておく

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
- タスクは `apt_sources` / `locale` タグで個別実行もできます（例: `--tags locale`）。

## 今後の拡張候補（未実装）
- SSH鍵配布・パスワード認証の無効化
- root以外の作業ユーザー作成
- 追加パッケージのインストール、systemdサービスの配置
- `ansible-vault` によるパスワード管理
