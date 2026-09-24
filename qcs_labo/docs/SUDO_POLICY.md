# sudo 方針ガイド（方針A / 方針B）

`qcsdemo` はユーザーにデモログインさせる想定のため、権限は継続的に見直します。
本ドキュメントは、現行の **方針A** と、将来の厳格運用 **方針B** の内容と、
その移行手順をまとめたものです。

## 方針の定義

すべて [`ansible/group_vars/all.yml`](../ansible/group_vars/all.yml) の
`users[].sudo_policy` と `sudo_policies` で制御します。

| 方針 | 種別 | 内容 | 主な用途 |
|------|------|------|----------|
| `unrestricted` | 制限なし | フル sudo（`ALL=(ALL) ALL`） | 管理者（`tisiadmin`） |
| `A` | ブロックリスト | フル sudo を許可しつつ、指定コマンド（`su` 等）を禁止 | デモ用（現行の `qcsdemo`） |
| `B` | ホワイトリスト | 指定コマンドのみ許可、それ以外は不可 | 厳格運用（将来の `qcsdemo`） |

sudoers はユーザーごとに `/etc/sudoers.d/<ユーザー名>` として配置され、
`visudo -cf` で構文検証してから適用されます（壊れた sudoers を防止）。

## 方針A の注意点（重要）

方針A は `su` を禁止しますが、`sudo -i` / `sudo bash` / `sudo -s` /
`sudo /bin/sh` など、root シェルを得る他の経路は残ります。
「`sudo su -` を打っても root になれない様子を見せる」程度のデモ用途では十分ですが、
**厳密に root 昇格を防ぐ目的には向きません**。厳格化する場合は方針B へ移行してください。

## 方針A → 方針B への移行手順

### ステップ1: qcsdemo が使うコマンドを洗い出す

デモで `qcsdemo` に許可したいコマンドを確定します。
初期値として `all.yml` の `sudo_policies.B.allowed_commands` に以下を定義済みです。

```yaml
sudo_policies:
  B:
    allowed_commands:
      - /usr/bin/dnf
      - /usr/bin/systemctl status *
      - /usr/bin/iperf3
      - /usr/bin/fio
      - /usr/bin/stress-ng
```

実際に必要なコマンドに合わせて追記・削除してください。
コマンドの絶対パスは対象サーバ上で `which <コマンド>` で確認できます。

### ステップ2: qcsdemo の方針を B に切り替える

`all.yml` の `qcsdemo` の `sudo_policy` を `A` から `B` に変更します。

```yaml
users:
  - name: qcsdemo
    ...
    sudo_policy: "B"   # ← A から B へ
```

### ステップ3: 適用

```bash
cd qcs_labo/ansible
# 変更内容の確認
ansible-playbook site.yml --tags users --check
# 適用
ansible-playbook site.yml --tags users
```

`/etc/sudoers.d/qcsdemo` がホワイトリスト定義に置き換わります。

### ステップ4: 検証

対象サーバに `qcsdemo` でログインし、想定どおりの制限になっているか確認します。

```bash
# 許可コマンドは実行できる
sudo dnf --version

# 許可外は拒否される（例）
sudo cat /etc/shadow        # -> 拒否されること
sudo su -                   # -> 拒否されること

# 現在の権限一覧を確認
sudo -l
```

## ロールバック（B → A に戻す）

`all.yml` の `qcsdemo` の `sudo_policy` を `A` に戻し、再適用します。

```bash
ansible-playbook site.yml --tags users
```

## 参考: 適用される sudoers の形（テンプレート）

生成ロジックは [`ansible/roles/users/templates/sudoers_user.j2`](../ansible/roles/users/templates/sudoers_user.j2) にあります。

- 方針A:
  ```
  Cmnd_Alias DENIED_QCSDEMO = /usr/bin/su, /bin/su
  qcsdemo ALL=(ALL) ALL, !DENIED_QCSDEMO
  ```
- 方針B:
  ```
  Cmnd_Alias ALLOWED_QCSDEMO = /usr/bin/dnf, /usr/bin/systemctl status *, ...
  qcsdemo ALL=(ALL) ALLOWED_QCSDEMO
  ```
