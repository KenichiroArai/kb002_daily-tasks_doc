# 03. 収集処理

[設計書の目次へ戻る](../backend.md)

## コマンド

[CarryoverCommandImpl](https://github.com/KenichiroArai/gr002_daily-tasks/blob/main/src/main/java/kmg/gr/gr002/dailytasks/carryover/presentation/command/impl/CarryoverCommandImpl.java) が引数を解釈します。収集するときは `ObjectProvider` で新しい [CarryoverService](https://github.com/KenichiroArai/gr002_daily-tasks/blob/main/src/main/java/kmg/gr/gr002/dailytasks/carryover/presentation/service/CarryoverService.java) を 1 回取り、`initialize` したあと、単位が続くあいだ同じインスタンスで `collect` と `aggregate` を繰り返します。単位の列と集計の追跡状態はサービスが持ちます。サービスは収集の設定をフィールドに持つため、実行のたびに新しいインスタンスを使います。受け付ける引数は次の 3 つだけです。不明な引数は `IllegalArgumentException` です。

```java
/* 収集と集計の実行 */
service.initialize(this.properties, this.full);

/* 次の単位があるか */
boolean hasNext = service.hasNext();

while (hasNext) {

    // 次の単位がある場合

    // その単位を収集する
    service.collect();

    // その単位までを集計して保存する
    service.aggregate();

    /* 次の単位があるか */
    hasNext = service.hasNext();

}
```

| 引数      | 定数                              | 動作                                       |
| --------- | --------------------------------- | ------------------------------------------ |
| `--full`  | `CommandOptionTypes.FULL`         | 全件モード。すべての Issue を解析し直す    |
| `--help`  | `CommandOptionTypes.HELP`         | 使い方を標準出力に出して終わる             |
| `-h`      | `CommandOptionTypes.HELP_SHORT`   | `--help` と同じ                            |

引数を付けない実行は差分モードです。`--help` または `-h` があるときは収集しません。

## 収集と集計

[CarryoverServiceImpl#collect](https://github.com/KenichiroArai/gr002_daily-tasks/blob/main/src/main/java/kmg/gr/gr002/dailytasks/carryover/application/service/impl/CarryoverServiceImpl.java) は、その単位の参照だけを取得、解析、保存します。`collect` は引数を取りません。設定とモードは、その前の `initialize` で役割ごとのサービスに入っています。戻り値は、その単位で解析して保存した Issue の件数です。続けて [CarryoverServiceImpl#aggregate](https://github.com/KenichiroArai/gr002_daily-tasks/blob/main/src/main/java/kmg/gr/gr002/dailytasks/carryover/application/service/impl/CarryoverServiceImpl.java) が、その単位を実行中の集計へ足し、その時点の `summary.json` を上書きし、結果をログに出します。単位の件数は `carryover.batchSize`（既定 365。1 未満のときは全件を 1 単位）です。取得・解析・保存の段階はコマンドからは呼びません。

最初の `hasNext` で、本文を持たない索引を作り、日付順に単位へ分けます。日付がない Issue は最後の単位にし、日別の行にはせず `latestIssue` だけ更新します。索引が空でも、空の集計を 1 回書くための単位を 1 つ作ります。前の単位の本文は次の単位へ残しません。

```mermaid
sequenceDiagram
  participant Command as CarryoverCommandImpl
  participant Service as CarryoverServiceImpl
  participant Collect as CarryoverCollectServiceImpl
  participant Aggregate as CarryoverAggregateServiceImpl
  Command->>Service: initialize
  loop hasNext
    Command->>Service: collect
    Service->>Collect: その単位だけ取得・解析・保存
    Command->>Service: aggregate
    Service->>Aggregate: 追跡状態へ追加して summary.json を上書き
  end
```

1. 設定を `CarryoverSourceModel` に変換し、収集サービスへ取得元、`recentCount`、全件モードかを渡す。集計サービスへ保存先を渡す。`batchSize` は統合サービスが読む
2. 最初の単位の前に、標準時間を 1 回読み、解析ロジックを初期化する。本文を持たない索引を作成順に取り、タイトルから日付を付ける。`recentFrom` は索引の番号から決める
3. 日付がある Issue を日付順に `batchSize` 件ずつ分け、日付がない Issue を最後の単位にする
4. 単位の各参照について、差分判定で解析が必要な Issue だけ本文を 1 件取得して解析し、その Issue を保存する。解析しない Issue は、その番号の `issues/NNNN.json` だけ読む
5. その単位の解析結果を、実行ごとの集計状態へ足す。日付がない Issue は最大の Issue 番号だけ更新する
6. その時点の `summary.json` を上書きする。項目の並びと `latestIssue` は、その単位までで確定する
7. モード、累計の取得件数、累計の解析件数、その時点の日数、累計の補完件数、時間を決められなかった件数をログに出す

Issue の一覧は実行のたびに先頭ページから取得します。一覧の応答に含まれる本文は索引では捨て、本文が要る Issue だけ後から 1 件取得します。解析と保存を省くかどうかは、次の差分判定で決めます。

### Issue の取得

[GitHubIssueApiImpl#fetchIssueRefs](https://github.com/KenichiroArai/gr002_daily-tasks/blob/main/src/main/java/kmg/gr/gr002/dailytasks/carryover/repository/api/github/GitHubIssueApiImpl.java) が、応答の件数が 100 未満になるまでページを進め、番号、タイトル、状態、`updatedAt` だけを残します。プルリクエストは除きます。本文が要る Issue は [GitHubIssueApiImpl#fetchIssue](https://github.com/KenichiroArai/gr002_daily-tasks/blob/main/src/main/java/kmg/gr/gr002/dailytasks/carryover/repository/api/github/GitHubIssueApiImpl.java) が 1 件取得します。変換は `GitHubIssueRefDto` と `GitHubIssueDto` が行います。

```mermaid
sequenceDiagram
  participant Collect as CarryoverCollectServiceImpl
  participant Logic as DailyTaskIssueLogicImpl
  participant Api as GitHubIssueApiImpl
  participant GitHub as GitHub API

  Collect->>Logic: fetchIssueRefs
  Logic->>Api: fetchIssueRefs
  loop ページごとに
    Api->>GitHub: GET issues
    GitHub-->>Api: JSON
    Api->>Api: GitHubIssueRefDto を作る
  end
  Api-->>Logic: GitHubIssueRefDto の一覧
  Logic-->>Collect: DailyTaskIssueRefModel の一覧
  loop 本文が要る Issue
    Collect->>Logic: fetchIssue
    Logic->>Api: fetchIssue
    Api->>GitHub: GET issues/番号
  end
```

### 解析結果の保存

今回解析した Issue ごとに、モデルから DTO、DTO から JSON 用 DTO へ変換して `issues/NNNN.json` へ書き出します。

```mermaid
sequenceDiagram
  participant Collect as CarryoverCollectServiceImpl
  participant Logic as CarryoverDataLogicImpl
  participant Dao as CarryoverDataDaoImpl
  participant IssueFile as JSONファイル

  Collect->>Logic: saveIssue
  Logic->>Logic: CarryoverIssueDto を作る
  Logic->>Dao: saveIssue
  Dao->>Dao: CarryoverIssueJsonDto を作る
  Dao->>IssueFile: issues/NNNN.json へ書き出す
```

## 差分判定

`needsUpdate` が、その Issue を解析し直すかを返します。実装は [CarryoverCollectServiceImpl](https://github.com/KenichiroArai/gr002_daily-tasks/blob/main/src/main/java/kmg/gr/gr002/dailytasks/carryover/application/service/impl/CarryoverCollectServiceImpl.java) です。

| 条件                                       | 結果           |
| ------------------------------------------ | -------------- |
| 全件モード                                 | 解析し直す     |
| 保存済みの JSON がない                     | 解析し直す     |
| Issue 番号が `recentFrom` 以上             | 解析し直す     |
| 保存済みの `updatedAt` と取得した値が違う  | 解析し直す     |
| 上記以外                                   | 保存済みを使う |

`recentFrom` は、取得した Issue 番号を大きい順に `recentCount` 件（既定は 10）取り、その中の最小の番号です。Issue が無い場合は 0 です。モードの意味（最新 10 件は毎回解析する、それより前は更新されたものだけ、集計は毎回作り直す）は [Issue 解析の仕様](../../carryover-spec.md) の「収集のモード」に書きます。

## 解析

[CarryoverParseLogicImpl#parse](https://github.com/KenichiroArai/gr002_daily-tasks/blob/main/src/main/java/kmg/gr/gr002/dailytasks/carryover/domain/logic/impl/CarryoverParseLogicImpl.java) が、1 件の Issue から `CarryoverIssueModel` を作ります。標準時間は `initialize` で保持したフィールドを使います。

- 日付はタイトルから取る
- 対象セクションのチェックリスト行を項目にする
- 時間表記がない行は標準時間で補完する
- 本文の `残：N` と解析件数を比べ、食い違う場合は警告ログを出す

対象セクション、時間表記、項目名の正規化、`declaredCount` の規則は [Issue 解析の仕様](../../carryover-spec.md) を参照してください。正規表現は `CarryoverParsePatternTypes`、日付書式などの固定文字列は `CarryoverParseTextTypes`、対象セクション名は `TargetSectionTypes` に置きます。

## 集計

[CarryoverAggregateLogicImpl#add](https://github.com/KenichiroArai/gr002_daily-tasks/blob/main/src/main/java/kmg/gr/gr002/dailytasks/carryover/domain/logic/impl/CarryoverAggregateLogicImpl.java) が、日付順に渡された 1 件を実行ごとの集計状態へ足します。一括の `aggregate` も、並べ替えてから同じ `add` を呼びます。追跡状態は集計ロジックのフィールドには置かず、実行ごとに作る累積モデルが持ちます。

1. 日付がある Issue を、日付の昇順、同じ日付なら Issue 番号の昇順に並べる
2. 対象セクションが最初に現れた Issue から集計を始める。それより前の Issue は日別の行にしない
3. 集計を始めたあとの Issue は、対象セクションが無い日も 0 件の日として残す
4. 各項目を、その日の合計、項目ごと、持ち越し元の月ごとに足す。持ち越し元の日付が無い行は月ごとの集計に含めない。月は `originDate` の先頭 7 文字（`YYYY-MM`）とする
5. 項目名は、延べ件数の多い順、同じ件数なら名前の昇順に並べる
6. `latestIssue` は、渡された解析結果の Issue 番号の最大値とする。解析結果が空なら 0 とする

`summary.json` の項目の意味は [Issue 解析の仕様](../../carryover-spec.md) を参照してください。

## Issue の取得

[GitHubIssueApiImpl](https://github.com/KenichiroArai/gr002_daily-tasks/blob/main/src/main/java/kmg/gr/gr002/dailytasks/carryover/repository/api/github/GitHubIssueApiImpl.java) が GitHub REST API から Issue を取得します。呼び出しの順序は、上の「Issue の取得」のシーケンス図です。接続設定は `initialize` でフィールドに保持し、一覧の取得と 1 件取得は番号以外の引数を取りません。

| 項目           | 値                                                                 |
| -------------- | ------------------------------------------------------------------ |
| 1 ページの件数 | 100。応答の件数が 100 未満になるまでページを進める                 |
| タイムアウト   | 30 秒                                                              |
| 成功           | HTTP ステータス 200                                                |
| 認証           | トークンがあるときだけ `Authorization` ヘッダを付ける              |
| 固定文字列     | URL、ヘッダ、空文字、応答の項目名は repository の `XxxTypes` に置く |

テストでは GitHub API へ実通信しません。`HttpClient` を生成するフィールドを差し替えます。
