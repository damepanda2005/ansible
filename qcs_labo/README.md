# QCS Labo - Rocky Linux 9 デモ環境

QCS（クラウド基盤）上に構築した **Rocky Linux 9** のデモ環境に対して、
初期構築（ユーザー登録・SELinux 停止・パッケージ導入・OS 最新化）と、
サーバ構成情報の収集（Markdown レポート）、パラメータシート（Excel）生成を行います。

## 対象サーバ

| インベントリ名 | IP | OS | 接続ユーザー |
|----------------|-----|-----|--------------|
| demo-linux02 | 172.25.23.93 | Rocky Linux 9.6 | admin |
| demo-linux03 | 172.25.23.94 | Rocky Linux 9.6 | admin |

## 実施内容（構築）

1. **ユーザー登録**
   - `qcsdemo`（デモログイン用 / sudo 可・`su` 禁止＝方針A）
   - `tisiadmin`（管理用 / フル sudo）
2. **SELinux 停止**（永続 `disabled`。再起動後に完全反映）
3. **パッケージ導入**（`iperf3`, `fio`, `stress-ng`。`stress-ng` 用に EPEL 有効化）
4. **OS 最新化**（`dnf update`。必要に応じ再起動要否を判定）

すべての設定値は [`ansible/group_vars/all.yml`](ansible/group_vars/all.yml) に集約しています
（設定値の唯一のソース）。

## ディレクトリ構成

```
qcs_labo/
├── README.md                        ← このファイル
├── ansible/
│   ├── ansible.cfg                  ← Ansible 設定
│   ├── requirements.yml             ← 必要コレクション
│   ├── site.yml                     ← 構築用プレイブック
│   ├── gather_info.yml              ← 構成情報収集プレイブック
│   ├── inventory/
│   │   └── hosts.ini                ← 対象サーバ定義
│   ├── group_vars/
│   │   └── all.yml                  ← 全パラメータ（設定値のソース）
│   └── roles/
│       ├── common/                  ← OS 確認・概要表示
│       ├── users/                   ← ユーザー登録 + sudo 設定
│       ├── selinux/                 ← SELinux 設定
│       ├── packages/                ← パッケージ導入（EPEL 含む）
│       ├── os_update/               ← OS 最新化
│       └── gather_info/             ← 構成情報収集 + MD レポート生成
└── docs/
    ├── generate_param_sheet.py      ← Excel パラメータシート生成スクリプト
    ├── SUDO_POLICY.md               ← sudo 方針 A→B 移行手順
    └── reports/                     ← 収集した構成情報レポート（自動生成）
```

## 前提（踏み台 = 制御ノード側）

- Ansible がインストールされていること
- 対象サーバ（QCS ノード）へ SSH 接続できること
- Python3（Rocky 9 は標準搭載）

### 認証方式

- **SSH ログイン**: `admin` / **パスワード認証**
  - 実行時に `-k`（`--ask-pass`）でパスワードを入力します（ファイルに保存しない運用）。
- **sudo**: パスワード未設定のため、`-K`（sudo パスワード入力）は不要です。
- パスワード認証には踏み台に **`sshpass`** が必要です。未導入なら次で導入します。

```bash
# sshpass の導入（Rocky/RHEL 系。EPEL が必要な場合あり）
sudo dnf install -y epel-release
sudo dnf install -y sshpass
```

```bash
# 必要コレクションの導入（初回のみ）
cd qcs_labo/ansible
ansible-galaxy collection install -r requirements.yml
```

## 使い方

> `-k` は SSH パスワードを実行時にプロンプト入力するオプションです。
> sudo はパスワード未設定のため `-K` は不要です。

### 1. 接続確認

```bash
cd qcs_labo/ansible
ansible demo -m ping -k
```

### 2. 構築（初期セットアップ）

```bash
# ドライラン（変更内容の確認のみ）
ansible-playbook site.yml --check -k

# 実適用
ansible-playbook site.yml -k

# 一部だけ適用（例: ユーザーのみ）
ansible-playbook site.yml --tags users -k
```

利用可能なタグ: `common`, `users`, `selinux`, `packages`, `os_update`

### 3. 構成情報レポートの生成

```bash
ansible-playbook gather_info.yml -k
# 出力: docs/reports/<ホスト名>_構成情報.md
```

### 4. パラメータシート（Excel）の生成

```bash
cd ../docs
python generate_param_sheet.py
# 出力: docs/QCS_Labo_パラメータシート.xlsx
```

`group_vars/all.yml` の値をそのまま反映するため、Ansible の適用値と
Excel の記載内容が常に一致します。

## パラメータの変更

設定を変えたいときは [`ansible/group_vars/all.yml`](ansible/group_vars/all.yml) を編集します。
ユーザー追加・パッケージ変更・SELinux 状態・OS 最新化の有無などをここで一元管理します。

## sudo 方針について

`qcsdemo` はデモログイン用のため、権限は見直し前提です。
現在は **方針A（ブロックリスト方式：`su` を禁止）** で運用しています。
将来的な厳格化（**方針B：ホワイトリスト方式**）への移行手順は
[`docs/SUDO_POLICY.md`](docs/SUDO_POLICY.md) を参照してください。
