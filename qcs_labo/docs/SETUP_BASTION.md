# 踏み台サーバ セットアップ手順（Rocky Linux 9.6）

踏み台サーバを **Ansible 制御ノード** として使うための初期セットアップ手順です。
Git・Ansible・sshpass を導入し、本リポジトリを取得して実行できる状態にします。

対象踏み台 OS: **Rocky Linux 9.6**

> パッケージはいずれも dnf の標準／EPEL リポジトリから最新版を導入します。
> バージョン指定は不要です。

## 1. リポジトリの有効化（EPEL）

`ansible-core` や `sshpass` は EPEL リポジトリを利用します。

```bash
sudo dnf install -y epel-release
sudo dnf makecache
```

## 2. Git / Ansible / sshpass の導入

```bash
# まとめて導入
sudo dnf install -y git ansible-core sshpass
```

| パッケージ | 用途 |
|-----------|------|
| `git` | リポジトリの取得（clone / pull） |
| `ansible-core` | Ansible 本体（プレイブック実行） |
| `sshpass` | パスワード認証での SSH 接続に必要 |

> 備考: `ansible-core` は最小構成の本体です。多数の同梱コレクションが欲しい場合は
> 代わりに `sudo dnf install -y ansible` を使う選択肢もありますが、
> 本プロジェクトは `requirements.yml` で必要コレクションを明示導入するため
> `ansible-core` で十分です。

## 3. バージョン確認

```bash
git --version
ansible --version
sshpass -V
```

`ansible --version` で `python version` が 3.x になっていることも確認してください
（Rocky 9 は標準で python3）。

### 確認済みの踏み台環境（参考）

踏み台 `qcs-zabbix-proxy` での実測値:

| ツール | バージョン |
|--------|-----------|
| git | 2.52.0 |
| ansible-core | 2.14.18 |
| sshpass | 1.09 |
| python | 3.9.25 |

> **重要**: `ansible --version` の `config file` が `/etc/ansible/ansible.cfg` と
> 表示される場合、プロジェクトの設定が使われていません。
> 本プロジェクトの `ansible.cfg` を有効にするため、コマンドは必ず
> **`qcs_labo/ansible/` ディレクトリ内で実行** してください
> （カレントディレクトリの `ansible.cfg` が優先されます）。
>
> なお `requirements.yml` は ansible-core 2.14 と互換のバージョン範囲に
> 固定しています。新しすぎるコレクションは 2.15+ を要求するため注意してください。

## 4. リポジトリの取得

```bash
# 任意の作業ディレクトリで
git clone <リポジトリURL>
cd <repo>/qcs_labo/ansible
```

更新を取り込むときは：

```bash
cd <repo>
git pull
```

## 5. 必要な Ansible コレクションの導入

本プロジェクトは `ansible.posix` / `community.general` を使用します。

```bash
cd <repo>/qcs_labo/ansible
ansible-galaxy collection install -r requirements.yml
```

## 6. 接続確認 → 実行

認証は `admin` / パスワード認証のため `-k` を付けます（sudo はパスワード未設定のため `-K` 不要）。

```bash
# 接続確認（昇格不要なので become 無効化）
ansible demo -m ping -k -e ansible_become=false

# 構築（ドライラン → 適用）。QCS ノードの sudo はパスワード必須のため -K も付ける
ansible-playbook site.yml --check -k -K
ansible-playbook site.yml -k -K

# 構成情報レポート生成（読み取り中心なので昇格なし）
ansible-playbook gather_info.yml -k -e ansible_become=false
```

## トラブルシューティング

- **`ansible` コマンドが見つからない**
  `ansible-core` が入っているか確認（`dnf list installed ansible-core`）。
  pip 導入した場合は `~/.local/bin` に PATH を通す。

- **パスワード認証で `sshpass` 関連のエラー**
  `sshpass` が未導入の可能性。手順2で導入する。

- **初回接続で host key の確認を求められる／失敗する**
  本プロジェクトは `inventory` と `ansible.cfg` でホストキー確認を無効化済み。
  それでも出る場合は `~/.ssh/known_hosts` の該当ホスト行を削除して再実行。

- **`epel-release` が見つからない**
  `sudo dnf install -y epel-release` の前に
  `sudo dnf install -y dnf-plugins-core` を実行し、リポジトリ状態を確認する。
