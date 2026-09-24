# Ansible プロジェクト（QCS 検証用）

このリポジトリは **Ansible 専用** のプロジェクトです。
目的ごとにフォルダを分け、その配下に Ansible のコード一式を配置します。

## ディレクトリ方針

```
ansible/                        ← リポジトリルート
├── README.md                   ← このファイル（プロジェクト全体の説明）
├── .gitignore
└── <目的フォルダ>/             ← 目的ごとに 1 フォルダ
    ├── ansible/                ← Ansible コード一式（playbook / roles / inventory ...）
    └── docs/                   ← その目的のドキュメント・成果物
```

## 目的フォルダ一覧

| フォルダ | 目的 | 対象 |
|----------|------|------|
| [`qcs_labo/`](qcs_labo/README.md) | QCS 上の Rocky Linux 9 デモ環境の初期構築（ユーザー登録 / SELinux 停止 / パッケージ導入 / OS 最新化）と構成情報の収集 | QCS 上の Rocky Linux 9 |

## 運用の全体像（Git 経由）

```
[このPC (Windows)]        [Git リポジトリ]        [踏み台 = Ansible 制御ノード]        [QCS ノード群]
   VS Code で編集   ──push──▶            ──clone/pull──▶   ansible-playbook 実行   ──SSH──▶  Rocky Linux 9
```

- 編集はこの PC（Windows）で行い、Git へ push します。
- 踏み台サーバへログインし、`git clone` / `git pull` してから `ansible-playbook` を実行します。
- Ansible は踏み台から各 QCS ノードへ SSH 接続して構成を適用します。

> 注: Windows は Ansible の制御ノードとして正式サポートされていません。
> 本プロジェクトでは **踏み台サーバを制御ノード** として運用します。

各目的フォルダの詳細は、それぞれの `README.md` を参照してください。
