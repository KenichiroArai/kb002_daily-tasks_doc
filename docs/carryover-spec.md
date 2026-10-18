# Issue 解析の仕様

収集ツール（[gr002_daily-tasks](https://github.com/KenichiroArai/gr002_daily-tasks)）が daily-tasks-git の Issue を解析し、JSON を出力するときの仕様です。
解析ルールを変更した場合は、本書と収集ツールの両方を更新し、全件モードで JSON を作り直します。

## 対象の Issue

- 対象リポジトリ: `https://github.com/KenichiroArai/daily-tasks-git/`（Issue #1 から最新まで）
- 日付は Issue タイトルの `YYYY年MM月DD日` から取得し、「その日の残」の日付とします

## 対象のセクション

| セクション    | 対象の Issue |
| ------------- | ------------ |
| `## 負債`     | #175〜#229   |
| `## 繰り越し` | #230〜#350   |
| `## 持ち越し` | #351〜       |

- 見出しから次の `##` までを対象とします
- `## ルーティン`、`## 汎用タスク`、`## 追加`、`## 先行` は対象外です

## 行の解析

- 未チェック（`- [ ]`）・チェック済み（`- [x]`）の両方を対象とし、`checked` フラグで区別します
- 行の形式は `項目名YYYY/MM/DD（時間表記）` です。日付は持ち越し元の日付（`originDate`）とします
- 時間表記は次のいずれかです。N は小数（例: `8.5`）を許容します

| 時間表記        | `minutesSource` |
| --------------- | --------------- |
| `残り時間：N分` | `parsed`        |
| `N分`           | `parsed`        |
| `残りN分`       | `parsed`        |
| `先行分残りN分` | `parsed`        |
| 時間表記なし    | `default`       |

- 時間表記がない行は、daily-tasks-git の `config/default-minutes.json` の項目ごとの標準時間で補完します
- 項目名は正規化します（末尾の `(` や `（15分）` などの混入を除去）
- 本文の `残：N` は検証用に `declaredCount` として保存し、解析件数と食い違う場合は警告ログを出します
- フォーマットの揺れに強い解析にし、想定外の行は落とさずに警告ログを出します

## 収集のモード

- Issue の一覧は毎回すべて取得します（100 件 / ページ）。解析・保存するかどうかはモードで決めます
- 差分モード（既定）: 保存済みの JSON（`docs/data/issues/NNNN.json` の `updatedAt`）を「どこまで処理したか」の記録として使います。Issue 番号が大きい順の最新 10 件（10 日分）は `updatedAt` に関係なく毎回解析し直して上書きします。それより前の Issue は、未取得の Issue と更新された Issue（編集やクローズを含む）だけを解析・保存します
- 全件モード（`--full`）: #1 から最新まで解析し直します。解析ルールや標準時間を変えた場合はこちらを使います
- 集計（`summary.json`）は、対象セクションが最初に現れた Issue（#175、2026-03-24）以降を毎回すべて作り直します

## 出力する JSON

出力先は daily-tasks-git の `docs/data/` です。Git で管理し、手で編集しません。

| ファイル                     | 内容                                 | 利用者                             |
| ---------------------------- | ------------------------------------ | ---------------------------------- |
| `docs/data/issues/NNNN.json` | Issue ごとの解析結果（4 桁ゼロ埋め） | 収集ツール（差分の判定）           |
| `docs/data/summary.json`     | 画面用の日別集計                     | 画面（mk002_daily-tasks_web-next） |

### issues/NNNN.json

```json
{
  "number": 360,
  "title": "2026年09月25日のタスク",
  "date": "2026-09-25",
  "state": "closed",
  "updatedAt": "2026-09-26T01:39:25Z",
  "sections": ["持ち越し"],
  "declaredCount": 108,
  "count": 108,
  "minutes": 2255.25,
  "items": [
    {
      "name": "国語",
      "originDate": "2026-06-14",
      "checked": true,
      "minutes": 15.0,
      "minutesSource": "parsed",
      "section": "持ち越し",
      "raw": "- [x] 国語2026/06/14（残り時間：15分）"
    }
  ]
}
```

### summary.json

```json
{
  "latestIssue": 378,
  "items": ["英語", "国語"],
  "days": [
    {
      "date": "2026-03-24",
      "issue": 175,
      "declaredCount": null,
      "total": {
        "count": 2,
        "minutes": 30.0,
        "checkedCount": 0,
        "checkedMinutes": 0.0
      },
      "byItem": {
        "英語": {
          "count": 2,
          "minutes": 30.0,
          "checkedCount": 0,
          "checkedMinutes": 0.0
        }
      },
      "byOriginMonth": {
        "2026-03": {
          "count": 2,
          "minutes": 30.0,
          "checkedCount": 0,
          "checkedMinutes": 0.0
        }
      }
    }
  ]
}
```

- 画面での検証（zod スキーマ）の詳細は [フロントエンド設計書の 02. データ設計](design/frontend/02-data.md) を参照してください
- 収集ツールでの型の変換は [バックエンド設計書の 02. データ設計](design/backend/02-data.md) を参照してください
- JSON の形式を変える場合は、収集ツール、画面の zod スキーマ、本書をあわせて更新し、全件モードで再生成できることを確認します
