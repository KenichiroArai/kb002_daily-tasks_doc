# AGENTS.md — daily-tasks（ドキュメント）

AI コーディングエージェント向けの作業ガイド。
Cursor / Codex / Claude Code など複数ツールで共通利用する。

本ドキュメントは、daily-tasks の設計書と横断的なドキュメントを書くときのルールをまとめる。

## プロジェクト概要

- **役割**: [daily-tasks-git](https://github.com/KenichiroArai/daily-tasks-git) を基軸とする daily-tasks のうち、プログラム部分（KMG シリーズの第 2 弾）に関するドキュメントを管理する
- **対象**: バックエンド（[gr002_daily-tasks](https://github.com/KenichiroArai/gr002_daily-tasks)）とフロントエンド（[mk002_daily-tasks_web-next](https://github.com/KenichiroArai/mk002_daily-tasks_web-next)）に関係するもの、およびそれらに横断的なもの
- **対象外**: プログラムに関係しない日々のタスクの運用（Issue の書き方など）は daily-tasks-git で管理する

## ディレクトリ構成

```text
docs/
  overview.md          # 全体構成（リポジトリの役割と関係、ローカルの配置、データの流れ、ワークフロー）
  carryover-spec.md    # Issue 解析の仕様（収集のモード、出力する JSON の形式）
  coding-rules.md      # Java と TypeScript に共通するコーディングルール
  design.md            # フロントエンド設計書の入口（目次）
  design/              # フロントエンド設計書の各章（01-architecture.md 〜 05-quality.md）
```

## ドキュメントの置き場所

- 複数のリポジトリにまたがる内容（全体構成、データの形式、共通ルール）と設計書は本リポジトリに置く
- 言語ごとの作業ルール（Java のパッケージ構成、TypeScript の構成ルールなど）は、各リポジトリの `AGENTS.md` に置く
- 同じ内容を複数の場所に書かない。詳細はどちらか一方に書き、もう一方からはリンクする

## 書き方のルール

- Markdown で書き、日本語の文章は「です・ます」調にする（`AGENTS.md` は「である」調）
- 設計書は各リポジトリの現状の実装を正として記述する。実装と食い違う箇所を見つけたら更新する
- 他のリポジトリのファイルへのリンクは、GitHub の URL（`https://github.com/KenichiroArai/<リポジトリ>/blob/main/<パス>`）で書く。本リポジトリ内のリンクは相対パスで書く
- 図は Mermaid で書く。ノードの ID に空白を入れず、色やスタイルは指定しない
- 新しい章を追加した場合は、`design.md` の目次や `README.md` の一覧も更新する

## 更新のきっかけ

| 変更                                              | 更新するドキュメント                                        |
| ------------------------------------------------- | ----------------------------------------------------------- |
| 解析ルール・収集のモード・JSON の形式（gr002）    | `docs/carryover-spec.md`、必要なら `docs/design/02-data.md` |
| 画面のコード・設定（mk002）                       | `docs/design.md` と `docs/design/` の全章を見直す           |
| ワークフロー・リポジトリの構成（daily-tasks-git） | `docs/overview.md`、`docs/design/05-quality.md`             |
| 共通のコーディングルール                          | `docs/coding-rules.md` と各リポジトリの `AGENTS.md`         |

## やってはいけないこと

- ソースコードや出力データ（`docs/data/` の JSON）を本リポジトリに置くこと
- 実装を確認せずに設計書を書き換えること
- シークレット（`GITHUB_TOKEN` など）をドキュメントに書くこと

## 順守

以上の内容を順守し、タスクを遂行してください。
