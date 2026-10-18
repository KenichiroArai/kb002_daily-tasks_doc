# kb002_daily-tasks_doc

日々のタスク（GitHub Issue、1 日 1 Issue）の「持ち越し」を集計・公開する daily-tasks の、設計書と横断的なドキュメントです。
[daily-tasks-git](https://github.com/KenichiroArai/daily-tasks-git) を基軸とし、プログラム部分（バックエンド・フロントエンド）に関係するドキュメントを置いています。

## ドキュメント

| ドキュメント                                     | 内容                                                                   |
| ------------------------------------------------ | ---------------------------------------------------------------------- |
| [全体構成](docs/overview.md)                     | リポジトリの役割と関係、ローカルの配置、データの流れ、ワークフロー     |
| [Issue 解析の仕様](docs/carryover-spec.md)       | 対象セクション、時間表記、収集のモード、出力する JSON の形式           |
| [共通のコーディングルール](docs/coding-rules.md) | Java と TypeScript に共通するルール                                    |
| [設計書](docs/design.md)                         | フロントエンドとバックエンドの設計書への入口                           |
| [フロントエンド設計書](docs/design/frontend.md) | 画面の全体構成、データ、画面、モジュール、品質・運用                   |
| [バックエンド設計書](docs/design/backend.md)     | 収集ツールの全体構成、データ、収集処理、モジュール、品質・運用         |

## 関連リポジトリ

| リポジトリ                                                                                | 役割                                                       |
| ----------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| [daily-tasks-git](https://github.com/KenichiroArai/daily-tasks-git)                       | 基軸。Issue、ワークフロー、設定ファイル、出力データ、Pages |
| [gr002_daily-tasks](https://github.com/KenichiroArai/gr002_daily-tasks)                   | 収集ツール（Java）                                         |
| [mk002_daily-tasks_web-next](https://github.com/KenichiroArai/mk002_daily-tasks_web-next) | 画面（Next.js + TypeScript）                               |
| kb002_daily-tasks_doc（本リポジトリ）                                                     | 設計書と横断的なドキュメント                               |
