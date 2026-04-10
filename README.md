# home-ansible

自宅の Raspberry Pi を管理するための Ansible リポジトリ。
監視、ホームオートメーション、自宅外からのアクセス環境を構成する。

## 🏗 Architecture & Inventory

| Hostname         | Model      | Role / Main Services      | Note                                                                          |
| ---------------- | ---------- | ------------------------- | ----------------------------------------------------------------------------- |
| raspberrypi0.lan | RPi Zero W | Sensor Node               | Node Exporter, SmartMeter Exporter, Sensor Exporter                           |
| raspberrypi3.lan | RPi 3B+    | Network Ingress & Monitor | Cloudflared, Alloy （Scraper & Log Receiver）, Slack Bot for Google Assistant |
| raspberrypi4.lan | RPi 4B     | Home Automation Hub       | SwitchBot Actions                                                             |

### Key Components

- Monitoring: Grafana Alloy（Agent）-> Grafana Cloud
  - SD カード保護: ログ（journald）はオンメモリで保持。さらに Alloy の WAL（Write Ahead Log）も `tmpfs` にマウントし、ディスク書き込み負荷を最小化。
  - Hub Nodes（Pi3, Pi4）: Grafana Alloy が稼働し、メトリクス送信とログ転送を行う。
  - Edge Nodes（Pi Zero）: リソース制約のため Alloy は配置せず、Exporter のみを稼働させる。
  - Scraping: raspberrypi3 上の Alloy が、Edge Nodes のメトリクスをリモートで収集（Pull）して Grafana Cloud へ送信する構成。
- Container Mgmt:
  - Pi3 / Pi4: Docker Compose (`geerlingguy.docker` + 自作 `docker_apps` ロール)
  - Pi Zero: Podman + Quadlet（リソース節約のため Systemd ネイティブ統合を利用）
- Networking: raspberrypi3 が `cloudflared` トンネルおよび `socat` プロキシの出入り口を担当。
- Automation: raspberrypi4 が重めの処理（SwitchBot 制御など）を集中的に担当。

---

## 🚀 Usage

### 前提条件 （Prerequisites）

- Ansible: `ansible-playbook` コマンドが実行可能であること。
- SOPS & age: 秘密情報の復号に必要。YubiKey または Mac の Touch ID（Secure Enclave）が設定されていること。

```bash
# 依存コレクションのインストール
ansible-galaxy install -r requirements.yml

```

### デプロイ実行 （Deployment）

全体適用:

```bash
ansible-playbook site.yml --diff

```

特定のタグのみ実行（高速化）:

```bash
# 全部のDockerコンテナの設定を更新したい場合
ansible-playbook site.yml -t docker_apps

# switchbot-actionsコンテナだけ更新したい場合
ansible-playbook site.yml -t switchbot-actions

```

Dry Run:

```bash
ansible-playbook site.yml --check --diff

```

---

## 🔐 Secrets Management （SOPS）

公開版のリポジトリには秘密情報を含めない。
`vars/secrets.example.yml` と `.sops.yaml.example` を元に、ローカル専用の `vars/secrets.sops.yml` と `.sops.yaml` を作成して使う。
これら 2 ファイルは `.gitignore` に含めてあり、Git には追加されない。

初回セットアップ:

```bash
cp .sops.yaml.example .sops.yaml
cp vars/secrets.example.yml vars/secrets.sops.yml
$EDITOR .sops.yaml
sops --encrypt --in-place vars/secrets.sops.yml
sops vars/secrets.sops.yml

```

以降の編集:

```bash
sops vars/secrets.sops.yml

```

主な管理対象:

- Grafana Cloud 認証情報（Prometheus / Loki）
- Cloudflare Tunnel Token
- Google Assistant API Credentials
- Slack App 認証情報（Bot Token / App Token）
- スマートメーター（Wi-SUN）接続情報（ID / Password）

---

## 📂 Directory Structure & Design Pattern

### `roles/docker_apps` （The Magic Part）

このリポジトリの核となるロール。`compose.yaml` / `docker-compose.yml` を静的に配置するのではなく、変数ベースで動的にファイルを収集しデプロイする設計になっている。

1. 探索ロジック:

- `host_vars/<host>/<app_name>/`（ホスト固有設定）
- `roles/docker_apps/templates/<app_name>/`（デフォルト設定）
- 上記 2 箇所をマージしてデプロイ先 (`/opt/stacks/<app_name>`) に配置する。

2. リスタート制御:

- 設定ファイル (`.env`, `compose.yaml` 等) に変更があった場合のみコンテナを再起動する。

### `roles/socat_proxy`

外部から特定の TCP 通信（443 等）を受け取り、バックエンドへ流す軽量プロキシ。

- 特徴: 常駐プロセスではなく Systemd Socket Activation を利用。アクセスが来るまで `socat` は起動しない。
- WOL 機能: 接続リクエストが来たら、まず親プロキシ（Mac 等）に Wake-on-LAN を飛ばし、起動するまで待機してから接続するスクリプトが含まれている。

---

## ➕ How to Add a New App

新しい Docker アプリ「myapp」を追加する手順:

1. デフォルトテンプレート作成（Optional）:
   `roles/docker_apps/templates/myapp/compose.yaml` を作成。
2. ホスト固有設定作成:
   `host_vars/raspberrypi4.lan/myapp/compose.override.yaml` や `.env.j2` を作成。
3. 有効化:
   `host_vars/raspberrypi4.lan/main.yml` の `docker_apps` リストに追加。

```yaml
docker_apps:
  - cloudflared
  - dockge
  - myapp # <--- ここに追加
```

---

## 🔧 Troubleshooting

### Q. Dockgeでスタックが見えない / 管理できない

`docker_apps` ロールは `/opt/stacks/` 配下にファイルを展開する。
Dockge もこのディレクトリを監視しているが、Ansible 管理下のファイルを Dockge の UI から編集すると次回の Ansible 実行時に上書きされるので注意。

- 設定変更 = Ansible のリポジトリを修正
- コンテナの再起動/ログ確認 = Dockge UI

### Q. Grafanaでログが出ない

Alloy の設定 (`config.alloy.j2`) を確認する。
Systemd のジャーナル転送設定が `host_vars/_base.alloy.j2` にある。

```bash
# Alloyのログ確認
ssh raspberrypi4.lan "journalctl -u alloy -f"

```

### Q. "Failed to decrypt" エラー

`.sops.yaml` がローカルに存在し、そこに書いた recipient と手元の鍵が対応しているか確認すること。
YubiKey が刺さっているか、あるいは `age-plugin-se` が利用可能な状態かも合わせて確認すること。
