# 03. 画面設計

[設計書の目次へ戻る](../frontend.md)

## 画面一覧

| ルート        | ファイル                                                                                                               | 内容                                           |
| ------------- | ---------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| `/`           | [app/page.tsx](https://github.com/KenichiroArai/mk002_daily-tasks_web-next/blob/main/src/app/page.tsx)                 | 全体概要（`dailyTasks/overview`）              |
| `/carryover`  | [app/carryover/page.tsx](https://github.com/KenichiroArai/mk002_daily-tasks_web-next/blob/main/src/app/carryover/page.tsx) | 持ち越しの推移（`CarryoverDashboard`）         |
| `/advance`    | [app/advance/page.tsx](https://github.com/KenichiroArai/mk002_daily-tasks_web-next/blob/main/src/app/advance/page.tsx)   | 先行の推移（advance 機能のダッシュボード）   |
| 該当なし      | [app/not-found.tsx](https://github.com/KenichiroArai/mk002_daily-tasks_web-next/blob/main/src/app/not-found.tsx)       | 「ページが見つかりません」とトップへのリンク   |

すべてのページは [app/layout.tsx](https://github.com/KenichiroArai/mk002_daily-tasks_web-next/blob/main/src/app/layout.tsx) の共通レイアウト（Header / PageContainer / Footer）の中に表示します。Header のナビゲーション（概要 `/`・持ち越し `/carryover/`・先行 `/advance/`）から 3 ルートを切り替えます。項目は `siteConfig.navItems` で定義し、現在のパスに対応するリンクをハイライトします（[config/site.ts](https://github.com/KenichiroArai/mk002_daily-tasks_web-next/blob/main/src/config/site.ts)）。

## 画面構成（トップページ）

```text
+------------------------------------------------------------+
| Header: 毎日タスク / 説明文                                  |
|         [概要] [持ち越し] [先行]                             |
+------------------------------------------------------------+
| 最新: 2026-10-07（#NNN）                                     |
| SummaryCards: [最新の残（件数）][最新の残り時間][期間平均]   |
|               [最大][最小]                                   |
| DashboardControls: 表示 / 指標 / 集計 / 対象 / 期間 + プリセット |
| ItemSelector: [すべて選択][すべて解除] □項目A □項目B ...     |
| CarryoverChart: 見出し / 補足 / グラフ                        |
| RankingTable: 項目ごとの集計（選択した期間）                 |
+------------------------------------------------------------+
| Footer: データ: <リポジトリ> の Issue ／ summary.json         |
+------------------------------------------------------------+
```

```mermaid
flowchart TB
  layout["RootLayout"] --> header["Header"]
  layout --> container["PageContainer"]
  layout --> footer["Footer"]
  header --> headerNav["HeaderNav"]
  headerNav -->|"Link"| overview["/"]
  headerNav -->|"Link"| carryoverPage["/carryover/"]
  headerNav -->|"Link"| advancePage["/advance/"]
  container --> dashboard["CarryoverDashboard"]
  dashboard -->|"loading / error"| panel["Panel（メッセージ）"]
  dashboard -->|"success"| view["DashboardView"]
  view --> meta["最新の日付と Issue 番号"]
  view --> cards["SummaryCards"]
  view --> controls["DashboardControls"]
  view --> selector["ItemSelector"]
  view --> chart["CarryoverChart"]
  view --> ranking["RankingTable"]
```

### 読み込みの状態ごとの表示

| 状態      | 表示                                                                                       |
| --------- | ------------------------------------------------------------------------------------------ |
| `loading` | パネルに「データを読み込んでいます…」                                                      |
| `error`   | 見出し「データを読み込めませんでした」と、エラーメッセージ（[02. データ設計](02-data.md)） |
| `success` | ダッシュボード本体（`DashboardView`）                                                      |

読み込みの状態は、マウントのたびに `loading` から始まり、取得と検証の結果で `success` または `error` になります。

```mermaid
stateDiagram-v2
  [*] --> loading
  loading --> success : 取得と検証に成功
  loading --> error : 取得または検証に失敗
```

### 各領域

| 領域              | 内容                                                                         |
| ----------------- | ---------------------------------------------------------------------------- |
| 最新情報          | `summary.days` の最後の日付と `latestIssue`。データが 0 日の場合は表示しない |
| SummaryCards      | 選択中の期間・項目・対象で計算した 5 枚のカード（下表）                      |
| DashboardControls | 表示条件の切り替え。期間の入力欄の範囲はデータの最初の日から最後の日まで     |
| ItemSelector      | 項目ごとのチェックボックスと色見本。「すべて選択」「すべて解除」ボタン       |
| CarryoverChart    | 表示の種類に応じたグラフ。見出しと補足は表示条件から作る                     |
| RankingTable      | 選択中の項目ごとの最新・平均・最大・最小・合計                               |

#### SummaryCards

期間内の最後の日を「最新」とします。期間内にデータがない場合は「データなし」のカード 1 枚だけを表示し、補足に「期間や項目の選択を見直してください」と出します。

| カード           | 値                                 | 補足                                               |
| ---------------- | ---------------------------------- | -------------------------------------------------- |
| 最新の残（件数） | 最新日の件数（指標に関係なく件数） | 日付と Issue 番号。前日がある場合は前日比（件数）  |
| 最新の残り時間   | 最新日の残り時間（時間）           | 同じ値を分で表示                                   |
| 期間平均（指標） | 選択した指標の 1 日あたりの平均    | 期間の日数                                         |
| 最大（指標）     | 選択した指標の期間内の最大         | 最大になった日付（同じ値が複数ある場合は最初の日） |
| 最小（指標）     | 選択した指標の期間内の最小         | 最小になった日付（同じ値が複数ある場合は最初の日） |

#### RankingTable

- 行は選択中の項目だけです
- 並び順は「最新の値の大きい順」、同じ場合は「平均の大きい順」です
- 値は指標に応じて丸めて書式化します（[04. モジュール設計](04-modules.md) の丸めと書式化）

## 表示条件

表示条件は [dailyTasks/carryover/model/options.ts](https://github.com/KenichiroArai/mk002_daily-tasks_web-next/blob/main/src/dailyTasks/carryover/model/options.ts) の `DashboardOptions` で表します。

| 項目 | キー          | 選択肢                                                                                              | 既定値    |
| ---- | ------------- | --------------------------------------------------------------------------------------------------- | --------- |
| 表示 | `view`        | 期間に応じた候補（下表）。並びの先頭は `item`                                                       | `item`    |
| 指標 | `metric`      | `minutes`（残り時間（分））、`hours`（残り時間（時間））、`count`（件数（残））                       | `minutes` |
| 集計 | `agg`         | `avg`（平均）、`sum`（合計）、`max`（最大）、`min`（最小）、`last`（期間末の値）                    | `avg`     |
| 対象 | `target`      | `all`（すべて）、`unchecked`（未チェックのみ）、`checked`（チェック済みのみ）                       | `all`     |
| 期間 | `from` / `to` | `YYYY-MM-DD`（両端を含む）。空文字は制限なし                                                        | 最新1日   |

### 表示の種類

| `view`        | ラベル             | グラフ                     | 横軸            | 集計の選択 | 項目の選択           | 1日の期間 | 複数日の期間 |
| ------------- | ------------------ | -------------------------- | --------------- | ---------- | -------------------- | --------- | ------------ |
| `item`        | 項目ごと           | 横棒グラフ（値の大きい順） | 項目            | 使う       | 行に反映             | 出す      | 出す         |
| `dailyItem`   | 日別推移           | 積み上げ面グラフ           | 日付            | 使わない   | 系列に反映           | 出さない  | 出す         |
| `monthlyItem` | 月ごと             | 積み上げ棒グラフ           | 月（`YYYY-MM`） | 使う       | 系列に反映           | 出さない  | 出す         |
| `originMonth` | 持ち越し元の月ごと | 棒グラフ（1 系列）         | 持ち越し元の月  | 使う       | 反映しない（全項目） | 出す      | 出す         |

- 「全体」の表示（旧 `daily` / `monthly`）は廃止しています
- セレクトの並びは上表の順です（先頭は常に「項目ごと」）
- 1日の期間（`from` と `to` が同じかつ空でない）では推移系（`dailyItem` / `monthlyItem`）を出しません。期間を1日に変えたとき、現在の表示が候補外なら `item` に戻します
- 集計を使う表示（`AGG_VIEWS`: `monthlyItem` / `item` / `originMonth`）以外では、「集計」のセレクトボックスを無効にします
- 集計は日ごとの残をもとに計算します。たとえば月ごとの平均は「その月の 1 日あたりの残」です
- `originMonth` は `byOriginMonth` を使うため、項目の選択に関係なく全項目が対象です

### グラフの見出しと補足

見出しは `表示：指標・集計（対象）` の形式です。指標はセレクトと同じ表記（件数は「件数（残）」）を使います。集計を使わない表示では `・集計` を省きます。

```text
日別推移：件数（残）（すべて）
月ごと：残り時間（時間）・平均（未チェックのみ）
```

補足は次を組み合わせて表示します。

| 条件                                 | 補足                                                                                          |
| ------------------------------------ | --------------------------------------------------------------------------------------------- |
| 常時                                 | 単位は件 / 分 / 時間です（選択中の指標に応じた単位）。                                        |
| `view` が `originMonth`              | 持ち越し元の月ごとの表示は項目の選択に関係なく全項目を対象にします。                          |
| 集計を使う表示（`originMonth` 以外） | 集計は＜選択中の集計＞（日ごとの残をもとに計算します。例: 平均は期間内の 1 日あたりの残）。 |
| 集計を使う `originMonth`             | 集計は＜選択中の集計＞です。                                                                  |
### ツールチップと凡例

- ツールチップは系列を値の大きい順に並べます
- 積み上げ表示で 2 系列以上ある場合は、ツールチップに合計も表示します
- 凡例は系列が 2 つ以上ある場合だけ表示します
- 横棒グラフ（`item`）では、棒ごとに項目の色を使い、ツールチップには項目名を表示します

### 数値軸の目盛

- 数値軸の下限は常に `0` です（縦グラフは Y 軸、横棒グラフは X 軸）
- 目盛の間隔は指標ごとに切りのいい値を選びます（件数は 1・2・5…、分は 10・20・50…、時間は 0.5・1・2…）
- 目盛の上限は、データの最大値を覆う最小の「切りのいい」値にします

### 期間

- 開始日と終了日を日付の入力欄で指定します（入力できる範囲はデータの最初の日から最後の日まで）
- プリセットボタンで期間をまとめて設定できます
- 初期の期間は「最新1日」（データの最後の日のみ。`from` と `to` が同じ）です

| ボタン   | 期間                                         |
| -------- | -------------------------------------------- |
| 最新1日  | データの最後の日のみ（両端を含む 1 日間）    |
| 直近30日 | 最後の日を終わりとする 30 日間（両端を含む） |
| 全期間   | データの最初の日から最後の日まで             |

### 項目の選択

- 初期状態ではすべての項目を選択します
- 選択期間内で、現在の指標・対象の値がすべて 0 の項目は、項目選択・グラフ・集計表に出しません（期間内のいずれか 1 日でも正なら表示します）
- 「すべて選択」は、上記の条件で表示されている項目だけを対象にします
- 選択した項目は、元の並び順（`summary.items` の順）を保ってグラフと表に使います
- 色は `summary.items` の並び順でパレットから割り当てます。選択や表示の有無を変えても項目の色は変わりません

## URL クエリによる初期値

URL のクエリ文字列で、表示条件の初期値を指定できます（[lib/queryOptions.ts](https://github.com/KenichiroArai/mk002_daily-tasks_web-next/blob/main/src/dailyTasks/carryover/lib/queryOptions.ts)）。

```text
https://<ユーザー>.github.io/daily-tasks-git/?view=monthlyItem&metric=hours
```

| キー     | 指定できる値                                            |
| -------- | ------------------------------------------------------- |
| `view`   | `item` / `dailyItem` / `monthlyItem` / `originMonth`    |
| `metric` | `minutes` / `hours` / `count`                           |
| `agg`    | `avg` / `sum` / `max` / `min` / `last`                  |
| `target` | `all` / `unchecked` / `checked`                         |

- 想定外の値は無視し、既定値を使います（廃止した `daily` / `monthly` も想定外として無視します）
- 初期期間は最新1日のため、クエリで `dailyItem` / `monthlyItem` を指定しても期間の候補外なら `item` に戻します
- 期間（`from` / `to`）と項目の選択はクエリで指定できません
- クエリは最初の描画時に 1 回だけ読み取ります。画面で条件を変えても URL は更新しません

## スタイル

- 全体の色・角丸は [styles/globals.css](https://github.com/KenichiroArai/mk002_daily-tasks_web-next/blob/main/src/styles/globals.css) の CSS 変数で定義します（`--bg`、`--panel`、`--border`、`--text`、`--muted`、`--accent`、`--radius`）
- `select`・`input[type='date']`・`button` の共通の見た目も `globals.css` で定義します
- 部品ごとのスタイルは各コンポーネントの `*.module.css` に置きます
