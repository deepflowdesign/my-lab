# iPhoneでClaude Codeを使うための自宅サーバー構築ガイド

構成の全体像は [README](README.md) を参照。

## サーバースペック

| 項目 | 内容 |
|------|------|
| 機種 | HiMeLE Quieter 4C |
| CPU | Intel N100（4コア/4スレッド、TDP 6W） |
| メモリ | 8GB LPDDR4x |
| ストレージ | 256GB eMMC + LVM |
| ファン | なし（完全ファンレス・無音） |
| OS | Ubuntu Server 24.04.1 LTS |

## セットアップ手順

### Phase 1: ブートUSB作成（MBPで実施）

#### 1. Ubuntu Server ISOダウンロード
```bash
# 公式サイトからダウンロード
# https://ubuntu.com/download/server
# ファイル: ubuntu-24.04.1-live-server-amd64.iso（約2.7GB）
```

#### 2. USBメモリにISOを書き込み
```bash
# USBメモリのデバイス名を確認
diskutil list

# USBメモリをアンマウント（例: /dev/disk4）
diskutil unmountDisk /dev/disk4

# ISOを書き込み（rdiskを使うと高速）
sudo dd if=~/Downloads/ubuntu-24.04.1-live-server-amd64.iso of=/dev/rdisk4 bs=1m status=progress

# 完了後、USBを取り出し
diskutil eject /dev/disk4
```

### Phase 2: Ubuntu Serverインストール

#### 1. BIOS設定
- USBメモリを挿してPCを起動
- F2/F12/DELなどでBIOSに入る
- Boot順序をUSBを最優先に変更
- Secure BootをOFFにする（必要な場合）

#### 2. Ubuntuインストーラー
1. 「Install Ubuntu Server」を選択
2. 言語: English（日本語でもOK）
3. キーボード: Japanese
4. インストールタイプ: **Ubuntu Server**（minimizedではない）
5. ネットワーク: 後で設定するのでスキップ可
6. プロキシ: 空欄（スキップ）
7. ミラー: デフォルトのまま
8. ストレージ: **Use an entire disk** → **Set up this disk as an LVM group**（暗号化は不要）
9. プロファイル設定:
   - 名前: （あなたの名前）
   - サーバー名: **home-server**（任意）
   - ユーザー名: **youruser**（任意）
   - パスワード: （任意）
10. Ubuntu Pro: **Skip for now**
11. SSH: **Install OpenSSH server** にチェック
12. Featured snaps: 何も選ばずContinue

インストール完了後、USBを抜いて再起動。

### Phase 3: ネットワーク設定

#### 1. 有線LANを接続してログイン
```bash
# ログイン
youruser / （パスワード）
```

#### 2. ネットワークインターフェース確認
```bash
ip link
# enp1s0 などのインターフェース名を確認
# 状態が DOWN の場合は以下を実行
sudo ip link set enp1s0 up
```

#### 3. netplan設定（DHCPで自動取得）
```bash
# 設定ファイルを作成
sudo nano /etc/netplan/00-installer-config.yaml
```

以下の内容を記述:
```yaml
network:
  version: 2
  ethernets:
    enp1s0:
      dhcp4: true
```

```bash
# パーミッション設定
sudo chmod 600 /etc/netplan/00-installer-config.yaml

# 設定を適用
sudo netplan apply

# IPアドレス確認
ip addr show enp1s0
# 192.168.x.x のようなIPが表示されればOK
```

### Phase 4: Tailscaleインストール

```bash
# インストール
curl -fsSL https://tailscale.com/install.sh | sh

# 起動・ログイン
sudo tailscale up
# 表示されるURLをブラウザで開いてログイン

# 確認
tailscale status
# home-server が表示されればOK
```

これで `ssh youruser@home-server` でどこからでも接続可能に。

### Phase 5: Syncthingインストール（サーバー側）

```bash
# インストール
sudo apt install syncthing

# サービス有効化・起動
sudo systemctl enable syncthing@youruser
sudo systemctl start syncthing@youruser

# Device IDを確認（後で使う）
syncthing --device-id
```

### Phase 6: Syncthingインストール（MBP側）

```bash
# インストール
arch -arm64 brew install syncthing

# サービス起動（Mac起動時に自動起動）
brew services start syncthing

# WebUIを開く
open http://localhost:8384
```

### Phase 7: Syncthing同期設定

#### MBP側（http://localhost:8384）
1. 「Add Remote Device」をクリック
2. サーバーのDevice ID（OP2LIWY...）を入力
3. Device Name: home-server
4. 「Save」

#### サーバー側（SSHポートフォワーディングでアクセス）
```bash
# MBPから
ssh -L 8385:localhost:8384 youruser@home-server
# ブラウザで http://localhost:8385 を開く
```

1. MBPからの接続リクエストを承認
2. 「Add Remote Device」でMBPを追加（Device IDはMBPのWebUIで確認）

#### フォルダ共有設定
1. MBP側で「Add Folder」
2. Folder Path: `~/development`
3. Folder ID: `development`
4. 「Sharing」タブで「home-server」にチェック
5. 「Save」

6. サーバー側で共有リクエストを承認
7. Folder Path: `~/development`（フォルダがなければ `mkdir -p ~/development`）

#### 同期開始
- 初回同期は時間がかかる（約7GBで10分程度）
- 「Up to Date」になれば完了
- 転送が始まらない場合は両方のSyncthingを再起動:
  ```bash
  # サーバー
  sudo systemctl restart syncthing@youruser

  # MBP
  brew services restart syncthing
  ```

### Phase 8: Node.jsインストール

```bash
# NodeSourceリポジトリ追加
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -

# Node.jsインストール
sudo apt install -y nodejs

# 確認
node --version  # v24.12.0
npm --version
```

### Phase 9: Claude Codeインストール

```bash
# グローバルインストール
sudo npm install -g @anthropic-ai/claude-code

# 確認
claude --version  # 2.0.76 (Claude Code)
```

## 使い方

### 外出先からSSH接続

#### MBPから
```bash
ssh youruser@home-server
```

#### iPhoneから（Blink Shell）
1. Tailscaleアプリを起動して接続
2. Blinkを開く
3. `ssh youruser@home-server` を入力

### Blink Shellの操作

| 操作 | ジェスチャー |
|------|-------------|
| 新しいタブ | 2本指でタップ |
| タブを閉じる | 2本指で上にスワイプ |
| タブ切り替え | 左右にスワイプ |

### Claude Codeを使う
```bash
# 接続
ssh youruser@home-server
cd ~/development/your-project

# 新規セッション開始
claude

# 直前のセッションを再開
claude --continue

# 過去のセッション一覧から選んで再開
claude --resume
```

SSH接続が切れてもClaude Code側にセッションが残っているため、再接続後に `--continue` や `--resume` で作業を継続できる。

#### ワンライナーで直接セッション再開
```bash
ssh -t youruser@home-server "claude --continue"
```
Blink Shellの履歴に入れておくと、接続が切れても一発で復帰できる。

### Syncthing同期の仕組み
- **リアルタイム同期**: ファイル保存後、数秒で同期開始
- **定期スキャン**: 1時間ごとに自動スキャン（見逃し防止）
- **双方向**: MBP ↔ サーバー どちらで変更しても同期される

## トラブルシューティング

### ネットワークに繋がらない
```bash
# インターフェースをUPにする
sudo ip link set enp1s0 up

# netplan設定を確認
cat /etc/netplan/00-installer-config.yaml

# 再適用
sudo netplan apply
```

### Syncthingが同期しない
```bash
# 両方で再起動
# サーバー
sudo systemctl restart syncthing@youruser

# MBP
brew services restart syncthing
```

### SSH接続できない
```bash
# Tailscaleの状態確認
tailscale status

# SSHサービス確認
sudo systemctl status ssh
```

## Tailscale IPアドレス

| デバイス | Tailscale IP |
|----------|--------------|
| home-server | 100.x.x.x |
| your-macbook | 100.x.x.x |
| your-iphone | 100.x.x.x |

※ `tailscale status` で確認できます

## 費用まとめ

### 初期費用
| アイテム | 価格 |
|----------|------|
| HiMeLE Quieter 4C（N100/8GB/256GB） | ¥39,999 |
| KIOXIA USBメモリ 32GB | ¥759 |
| **合計** | **¥40,758** |

### 月額・年額費用
| アイテム | 価格 |
|----------|------|
| Claude Max（サブスク） | $200/月 |
| Blink Shell（iOSアプリ） | $19.99/年 |
| Ubuntu Server | 無料 |
| Tailscale | 無料 |
| Syncthing | 無料 |

---

## MBP側のSSH設定

サーバーにパスワードなしで接続できるよう設定する。

#### 1. ~/.ssh/config にホスト設定を追加

```bash
cat >> ~/.ssh/config << 'EOF'

Host home-server
  HostName 100.x.x.x
  User youruser
EOF
```

#### 2. サーバーのホストキーをknown_hostsに追加

```bash
ssh-keyscan -t ed25519 100.x.x.x >> ~/.ssh/known_hosts
```

#### 3. サーバー側に公開鍵を登録

サーバーにログインして以下を実行：

```bash
echo '（あなたの公開鍵）' >> ~/.ssh/authorized_keys
```

公開鍵は `cat ~/.ssh/id_ed25519.pub` で確認できます。

#### 4. SSH鍵をキーチェーンに登録

MBP側で以下を実行（パスフレーズを入力）：

```bash
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

これで `ssh home-server` でパスフレーズなしに接続可能になる。

---

## SSH接続安定化（オプション）

iPhoneからBlink ShellでSSH接続すると頻繁に切断される問題への対策。

### サーバー側のKeepAlive設定

`/etc/ssh/sshd_config` に以下を設定：

```
ClientAliveInterval 30
ClientAliveCountMax 3
```

- 30秒ごとにサーバーからクライアントにpingを送信
- 3回失敗で切断

設定後、sshdを再起動：

```bash
sudo systemctl restart ssh
```

### クライアント側（Blink）のKeepAlive設定

`~/.ssh/config` に追加：

```
Host *
  ServerAliveInterval 15
  ServerAliveCountMax 4
```

---

## セキュリティ強化（推奨）

Tailscaleを使っていれば外部からの直接攻撃リスクは低いですが、以下の設定を入れておくとより安心です。

### 1. SSHのパスワード認証を無効化

SSHキー認証のみに制限します。

```bash
sudo nano /etc/ssh/sshd_config
```

以下の2行を変更（コメントアウトされていたら外す）：

```
PermitRootLogin no
PasswordAuthentication no
```

設定を反映：

```bash
sudo systemctl restart ssh
```

**注意**: この設定を行う前に、SSHキー認証でログインできることを確認してください。パスワード認証を無効化した後にキー認証が失敗すると、サーバーにログインできなくなります。

### 2. ファイアウォール設定（ufw）

Tailscale IP（100.64.0.0/10）からのみアクセスを許可します。

```bash
# デフォルトポリシー設定
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Tailscale IPからのSSHを許可
sudo ufw allow from 100.64.0.0/10 to any port 22

# Tailscale IPからのSyncthingを許可
sudo ufw allow from 100.64.0.0/10 to any port 8384   # WebUI
sudo ufw allow from 100.64.0.0/10 to any port 22000  # 同期

# ファイアウォール有効化
sudo ufw enable

# 状態確認
sudo ufw status
```

### 3. 自動セキュリティアップデート

セキュリティパッチを自動適用します。

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

設定確認：

```bash
cat /etc/apt/apt.conf.d/20auto-upgrades
# 以下が表示されればOK
# APT::Periodic::Update-Package-Lists "1";
# APT::Periodic::Unattended-Upgrade "1";
```

### セキュリティ設定まとめ

| 設定 | 効果 |
|------|------|
| PermitRootLogin no | rootでの直接ログインを禁止 |
| PasswordAuthentication no | パスワード認証を禁止（キー認証のみ） |
| ufw | Tailscale IP以外からのアクセスをブロック |
| unattended-upgrades | セキュリティパッチ自動適用 |
