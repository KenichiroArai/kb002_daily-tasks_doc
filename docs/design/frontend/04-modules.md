# 04. モジュール設計

[設計書の目次へ戻る](../frontend.md)

## 処理の全体像

`CarryoverDashboard` がデータを読み込み、`DashboardView` が表示条件の状態を持ちます。表示用のデータはすべて `lib/` の純粋関数で作り、`useMemo` でメモ化してから各コンポーネントに渡します。

```mermaid
flowchart TB
  subgraph hooksLayer [hooks]
    useCarryoverSummary --> useAsync
    useQueryOptions --> parseQueryOptions
    useDashboardOptions --> dashboardReducer
  end
  subgraph libLayer [lib]
    filterDays
    assignColors
    itemsWithValueInPeriod
    orderSelectedItems
    buildSummaryCards
    buildChartSpec
    buildChartTitle
    buildChartNote
    buildRanking
  end
  useAsync --> fetchSummary
  fetchSummary --> summarySchema
  DashboardView --> useQueryOptions
  DashboardView --> useDashboardOptions
  DashboardView --> libLayer
  DashboardView --> SummaryCards
  DashboardView --> DashboardControls
  DashboardView --> ItemSelector
  DashboardView --> CarryoverChart
  DashboardView --> RankingTable
```

`DashboardView` での計算の順序は次のとおりです。

1. `assignColors(summary.items)`: 項目ごとの色（全項目基準。期間や表示の有無で色は変えない）
2. `filterDays(summary.days, from, to)`: 期間内の日別データ
3. `itemsWithValueInPeriod(days, summary.items, metric, target)`: 期間内で値が正の項目（`ItemSelector` の母集団）
4. `orderSelectedItems(visibleItems, selected)`: 表示対象のうち選択中の項目（元の並び順）
5. `buildSummaryCards` / `buildChartSpec` / `buildRanking`: 2 と 4 から表示用のデータを作る（`buildSummaryCards` は `{ days, items, options }` を渡す）
6. `buildChartTitle` / `buildChartNote`: 表示条件からグラフの見出しと補足を作る

### 表示条件の変更

表示条件を変えると、`useDashboardOptions` が reducer へ渡し、`DashboardView` が `useMemo` で `lib/` の関数を呼び直してから、カード・グラフ・表へ渡します。

```mermaid
sequenceDiagram
  participant User as ユーザー
  participant Controls as DashboardControls
  participant Selector as ItemSelector
  participant Options as useDashboardOptions
  participant Reducer as dashboardReducer
  participant View as DashboardView
  participant Lib as lib
  participant Parts as カード・グラフ・表

  alt 表示・指標・集計・対象・期間
    User->>Controls: 条件を変更
    Controls->>Options: setOptions / setPeriod / applyPreset
  else 項目の選択
    User->>Selector: 選択を変更
    Selector->>Options: toggleItem / selectAll / selectNone
  end
  Options->>Reducer: dispatch
  Reducer-->>Options: 新しい状態
  Options-->>View: options と selected
  View->>Lib: useMemo で表示用データを作る
  Lib-->>View: cards / spec / ranking
  View->>Parts: props を渡す
```

## 状態管理

### 読み込みの状態

[shared/types/loadState.ts](https://github.com/KenichiroArai/mk002_daily-tasks_web-next/blob/main/src/shared/types/loadState.ts) の `LoadState<T>` で表します。

```ts
type LoadState<T> =
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: Error };
```

### 画面の状態

[hooks/dashboardReducer.ts](https://github.com/KenichiroArai/mk002_daily-tasks_web-next/blob/main/src/dailyTasks/carryover/hooks/dashboardReducer.ts) の `DashboardState` で表し、`useReducer` で管理します。

| 項目       | 型                    | 説明           |
| ---------- | --------------------- | -------------- |
| `options`  | `DashboardOptions`    | 表示条件       |
| `selected` | `ReadonlySet<string>` | 選択中の項目名 |

初期状態は `createInitialState(summary, initialOptions)` で作ります。優先順位は「URL クエリの値」>「最新1日の期間」>「`DEFAULT_OPTIONS`」です。`DEFAULT_OPTIONS` の既定は `view: item`・`metric: minutes` です。項目はすべて選択します。期間に合わない `view` は `clampView` で `item` に戻します（`setOptions` / `setPeriod` でも同様）。

| アクション   | 引数                               | 処理                              |
| ------------ | ---------------------------------- | --------------------------------- |
| `setOptions` | `patch: Partial<DashboardOptions>` | 表示条件の一部を上書きする        |
| `setPeriod`  | `period: Period`                   | 期間（`from` / `to`）を上書きする |
| `toggleItem` | `item: string`                     | 項目の選択を切り替える            |
| `selectAll`  | `items: readonly string[]`         | 指定した項目をすべて選択する      |
| `selectNone` | なし                               | 選択をすべて解除する              |

reducer は状態を直接変更せず、常に新しいオブジェクト（`selected` は新しい `Set`）を返します。

#### reducer を `hooks/` に置く理由

- `useDashboardOptions` が `useReducer` に渡す専用の部品なので、同じ `hooks/` に置きます。
- reducer は React のフックを使わない純粋関数です。フックを動かさずに、入力と出力だけで単体テストできます。
- `useDashboardOptions` 専用の内部部品なので、`hooks/index.ts` では公開しません。
- `lib/` は複数の場所から使う計算を置く場所です。reducer は `hooks/` の中でしか使わないため、`lib/` には置きません。

## フック

| フック                | 場所                                                                                                                                                      | 役割                                                                                                                                                       |
| --------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `useAsync`            | [shared/hooks/useAsync.ts](https://github.com/KenichiroArai/mk002_daily-tasks_web-next/blob/main/src/shared/hooks/useAsync.ts)                            | 非同期処理をマウント時に実行し `LoadState` を返す。loader に `AbortSignal` を渡し、アンマウント時に中断して結果を捨てる。Error 以外の失敗は `Error` に包む |
| `useCarryoverSummary` | [hooks/useCarryoverSummary.ts](https://github.com/KenichiroArai/mk002_daily-tasks_web-next/blob/main/src/dailyTasks/carryover/hooks/useCarryoverSummary.ts) | `useAsync` で `fetchSummary` に `signal` を渡して summary.json を読み込む。loader はモジュールの定数にして参照を安定させる                                 |
| `useQueryOptions`     | [hooks/useQueryOptions.ts](https://github.com/KenichiroArai/mk002_daily-tasks_web-next/blob/main/src/dailyTasks/carryover/hooks/useQueryOptions.ts)         | 初回の描画時に 1 回だけ URL クエリを読み取る。`window` がない環境では空の条件を返す                                                                        |
| `useDashboardOptions` | [hooks/useDashboardOptions.ts](https://github.com/KenichiroArai/mk002_daily-tasks_web-next/blob/main/src/dailyTasks/carryover/hooks/useDashboardOptions.ts) | reducer を包み、`options`・`selected` と操作関数を返す。reducer（`dashboardReducer`）は内部用で `hooks/index.ts` から公開しない                                                                                                     |

`useDashboardOptions` が返す操作関数は次のとおりです。すべて `useCallback` で参照を安定させています。

| 関数          | 内容                                                         |
| ------------- | ------------------------------------------------------------ |
| `setOptions`  | 表示条件の一部を変更する                                     |
| `setPeriod`   | 期間を変更する                                               |
| `applyPreset` | 日数（0 は全期間、1 は最新1日）から `presetPeriod` で期間を作って設定する |
| `toggleItem`  | 項目の選択を切り替える                                       |
| `selectAll`   | 引数で渡した項目をすべて選択する（呼び出し側は `visibleItems`） |
| `selectNone`  | 選択をすべて解除する                                         |

## lib（純粋関数）

計算ロジックは `dailyTasks/carryover/lib/` の純粋関数にまとめ、コンポーネントから切り離します。外部の状態や DOM に依存しないため、Vitest で単体テストします。

### calc.ts（基本の計算）

| 関数                   | 内容                                                                                       |
| ---------------------- | ------------------------------------------------------------------------------------------ |
| `statValue`            | `Stat` から指標と対象に応じた値を取り出す。`Stat` がない場合は 0。`hours` は分を 60 で割る |
| `dayItemValue`         | ある日のある項目の値（`byItem` に項目がない場合は 0）                                      |
| `dayTotal`             | ある日の、指定した項目の合計値                                                             |
| `aggregate`            | 値の一覧を `avg` / `sum` / `max` / `min` / `last` で集計する。空の場合は 0                 |
| `roundMetric`          | 丸め。`hours` は小数 2 桁、それ以外は 1 桁                                                 |
| `formatMetric`         | 書式化（日本語ロケール）。`minutes` は整数、それ以外は小数 1 桁                            |
| `formatMetricWithUnit` | 丸めと書式化を行い、単位（件 / 分 / 時間）を付ける                                         |
| `groupByMonth`         | 日別データを月（`YYYY-MM`）ごとにまとめる。戻りは `DaysByMonth`（出現順を保つ）            |
| `filterDays`           | 期間（両端を含む）で日別データを絞り込む。空文字は制限なし                                 |
| `assignColors`           | 項目の並び順にパレットの色を割り当てる。戻りは `ItemColors`。数を超えたら先頭から繰り返す |
| `orderSelectedItems`     | 元の並び順を保ったまま、選択した項目だけを返す                                             |
| `itemsWithValueInPeriod` | 期間内で、指標と対象の値が 1 日でも正の項目だけを返す（元の並び順を保つ）                   |

対象（`target`）ごとの値の取り出し方は次のとおりです。

| `target`    | 件数                   | 時間                       |
| ----------- | ---------------------- | -------------------------- |
| `all`       | `count`                | `minutes`                  |
| `unchecked` | `count - checkedCount` | `minutes - checkedMinutes` |
| `checked`   | `checkedCount`         | `checkedMinutes`           |

### chartData.ts（グラフの定義）

`buildChartSpec(days, items, colors, options)` が、表示の種類ごとの作成関数を呼び分けて `ChartSpec` を返します。

```ts
type ChartSpec = {
  kind: "area" | "bar";
  stacked: boolean;
  horizontal: boolean;
  rows: ChartRow[]; // 横軸の 1 目盛りごとの行。label と系列キーごとの値
  series: ChartSeries[]; // key（s0, s1, ...）、label、color
};
```

- 系列のキーは項目名ではなく連番（`s0`、`s1`、...）にします。Recharts の `dataKey` は `.` をパスとして扱うため、項目名をそのまま使うと値を取り出せない場合があるためです
- 値は `roundMetric` で丸めてから行に入れます
- 1 系列の表示では `PRIMARY_COLOR`（`originMonth` は `ORIGIN_MONTH_COLOR`）を、項目ごとの表示では項目の色を使います
- `item` は値の大きい順に並べ、行ごとに項目の色（`row.color`）を持たせます
- `originMonth` の横軸は、期間内に現れた持ち越し元の月を昇順に並べたものです

### chartText.ts（グラフの文言）

| 関数              | 内容                                                                                         |
| ----------------- | -------------------------------------------------------------------------------------------- |
| `buildChartTitle` | グラフの見出し。指標は `METRIC_OPTION_LABELS`（[03. 画面設計](03-screen.md)）                |
| `buildChartNote`  | グラフの補足。単位（`UNIT_LABELS`）を常に出し、集計対象の表示では選択中の集計名も出す       |

### chartAxis.ts（数値軸の目盛）

| 関数              | 内容                                                                                         |
| ----------------- | -------------------------------------------------------------------------------------------- |
| `maxChartValue`   | グラフの数値の最大。積み上げなら行ごとの合計の最大、それ以外は系列ごとの最大                 |
| `buildValueAxis`  | 指標と最大値から、下限 0・切りのいい間隔の `domain` / `ticks` を返す（[03. 画面設計](03-screen.md)） |

### summaryCards/（サマリカード）

`buildSummaryCards({ days, items, options })` が各カード用の関数を呼び、`SummaryCardData[]`（ラベル・値・補足）を返します。期間内にデータがなければ「データなし」の 1 枚だけです。各カード関数には必要な引数だけを渡します（件数・時間カードは `target`、平均・最大・最小は `metric` と `target`）。カードを増やすときはファイルと関数を追加し、オーケストレータに並べます。

| ファイル               | 関数                     | 内容                         |
| ---------------------- | ------------------------ | ---------------------------- |
| `buildSummaryCards.ts` | `buildSummaryCards`      | 空データの判定とカードの並べ |
| `emptyCard.ts`         | `buildEmptyCard`         | データなしカード             |
| `latestCountCard.ts`   | `buildLatestCountCard`   | 最新の残（件数）             |
| `latestHoursCard.ts`   | `buildLatestHoursCard`   | 最新の残り時間               |
| `periodAverageCard.ts` | `buildPeriodAverageCard` | 期間平均                     |
| `maxCard.ts`           | `buildMaxCard`           | 最大                         |
| `minCard.ts`           | `buildMinCard`           | 最小                         |

### その他

| ファイル          | 関数                 | 内容                                                                                                      |
| ----------------- | -------------------- | --------------------------------------------------------------------------------------------------------- |
| `period.ts`       | `presetPeriod`       | 最新日を終わりとする直近 N 日間の期間。N が 0 なら全期間、1 なら最新1日。データがなければ空の期間。日付は UTC で計算する |
| `views.ts`        | `isSingleDayPeriod`  | 開始と終了が同じ（かつ空でない）とき 1 日の期間とみなす                                                   |
|                   | `availableViews`     | 期間に応じた表示の候補（1 日は `item` / `originMonth`、複数日は `MULTI_DAY_VIEWS`）                       |
|                   | `clampView`          | 期間の候補外の表示を `item` に戻す                                                                        |
| `queryOptions.ts` | `parseQueryOptions`  | クエリ文字列から `view` / `metric` / `agg` / `target` を読み取る。想定外の値は無視する                    |
| `ranking.ts`      | `buildRanking`       | 項目ごとの最新・平均・最大・最小・合計。最新の大きい順、同じなら平均の大きい順                            |

## コンポーネント

### 機能のコンポーネント（dailyTasks/carryover/components）

| コンポーネント       | 主な props                                                                       | 責務                                                                         |
| -------------------- | -------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| `CarryoverDashboard` | なし                                                                             | summary.json を読み込み、状態に応じてメッセージか `DashboardView` を表示する |
| `DashboardView`      | `summary`                                                                        | 表示条件の状態を持ち、表示用のデータを作って各部品に渡す                     |
| `SummaryCards`       | `cards`                                                                          | サマリカードを並べる（`aria-label="サマリ"`）                                |
| `DashboardControls`  | `options`、`minDate`、`maxDate`、`onOptionsChange`、`onPeriodChange`、`onPreset` | 表示・指標・集計・対象のセレクトボックス、期間の入力欄、プリセットボタン。表示の選択肢は `availableViews` で期間に応じて切り替える |
| `ItemSelector`       | `items`、`colors`、`selected`、`onToggle`、`onSelectAll`、`onSelectNone`         | 項目のチェックボックスと一括選択・解除                                       |
| `CarryoverChart`     | `title`、`note`、`spec`、`metric`                                                | `ChartSpec` から Recharts の面グラフ・棒グラフ・横棒グラフを描画する。数値軸の `domain` / `ticks` は `buildValueAxis` で決める |
| `ChartTooltip`       | `active`、`label`、`entries`、`metric`、`stacked`                                | 値の大きい順のツールチップ。積み上げで 2 系列以上なら合計も出す              |
| `RankingTable`       | `rows`、`colors`、`metric`                                                       | 項目ごとの集計表                                                             |

コンポーネントは表示とイベントの受け渡しだけを行い、計算は `lib/` の関数に任せます。状態を持つのは `DashboardView`（`useDashboardOptions`）だけです。

### 共通の部品（shared/components）

| コンポーネント  | 場所                   | props                                                | 内容                                                       |
| --------------- | ---------------------- | ---------------------------------------------------- | ---------------------------------------------------------- |
| `Panel`         | `ui/Panel`             | `title?`、`note?`、`className?`、`children`          | 枠付きの区画。見出しと補足は任意                           |
| `Select`        | `ui/Select`            | `label`、`value`、`options`、`onChange`、`disabled?` | ラベル付きのセレクトボックス（値の型をジェネリクスで保つ） |
| `Button`        | `ui/Button`            | `button` 要素の属性                                  | 共通のボタン。`type` の既定値は `button`                   |
| `Card`          | `ui/Card`              | `label`、`value`、`sub?`                             | ラベル・値・補足を縦に並べるカード                         |
| `DateRange`     | `ui/DateRange`         | `from`、`to`、`min?`、`max?`、`onChange`             | 開始日と終了日の入力欄                                     |
| `Swatch`        | `ui/Swatch`            | `color`                                              | 系列の色を示す小さな四角（`aria-hidden`）                  |
| `Header`        | `layout/Header`        | `title`、`description?`                              | ページ上部の見出し                                         |
| `Footer`        | `layout/Footer`        | `prefix?`、`links`                                   | ページ下部のリンク一覧（`／` 区切り）                      |
| `PageContainer` | `layout/PageContainer` | `children`                                           | 本文（`main` 要素）の幅と余白をそろえる                    |

### 共通の関数（shared/lib）

| 関数           | 内容                                                                      |
| -------------- | ------------------------------------------------------------------------- |
| `assetPath`    | basePath を付けた静的ファイルの URL（[01. 全体構成](01-architecture.md)） |
| `roundTo`      | 指定した小数の桁数で四捨五入する                                          |
| `formatNumber` | 日本語ロケールで書式化する。整数以外は指定した桁数に丸める                |

## 定数（dailyTasks/carryover/constants）

| ファイル     | 定数                   | 内容                                                     |
| ------------ | ---------------------- | -------------------------------------------------------- |
| `labels.ts`  | `VIEW_LABELS`          | 表示の種類のラベル（項目ごと / 日別推移 / 月ごと / 持ち越し元の月ごと） |
|              | `METRIC_LABELS`        | 指標のラベル（グラフの軸・カード用。件数は「件数」）                   |
|              | `METRIC_OPTION_LABELS` | 指標のラベル（セレクト・グラフ見出し用。件数は「件数（残）」）         |
|              | `AGG_LABELS`           | 集計方法のラベル                                                       |
|              | `TARGET_LABELS`        | 対象のラベル                                                           |
|              | `UNIT_LABELS`          | 単位（件 / 分 / 時間）                                                 |
|              | `PERIOD_PRESETS`       | 期間のプリセット（最新1日 / 直近30日 / 全期間）                        |
| `palette.ts` | `PALETTE`              | 系列の色（16 色）                                        |
|              | `PRIMARY_COLOR`        | 1 系列の表示の色（`PALETTE[0]`）                         |
|              | `ORIGIN_MONTH_COLOR`   | 持ち越し元の月ごとの表示の色（`PALETTE[4]`）             |

ラベルは `Record<View, string>` のように選択肢の型をキーにしているため、選択肢を追加するとラベルの定義漏れが型エラーになります。
