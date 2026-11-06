# 05. 品質・運用

[設計書の目次へ戻る](../backend.md)

## テスト

テストは JUnit 5 で、`src/test/java` に main と同じパッケージ構成で置きます。private のメソッドやフィールドは、テスト用の `testutil/ReflectionTestUtil` から標準のリフレクションで呼びます。GitHub API への実通信は行いません。

| 対象                         | 内容                                                                 |
| ---------------------------- | -------------------------------------------------------------------- |
| ロジック、サービス、DAO、API | 正常系、準正常系、異常系をメソッド単位で検証する                     |
| ログ                         | Logback の `ListAppender` で、出たメッセージを 1 行ずつ検証する      |
| 層のルール                   | `ArchitectureTest`（ArchUnit）が依存の向きなどを検査する             |
| カバレッジ                   | JaCoCo が行と分岐の 100% を要求する。未満だと `mvn test` は失敗する  |

テストメソッドの名前、Javadoc、実装順序の詳細は [gr002_daily-tasks の AGENTS.md](https://github.com/KenichiroArai/gr002_daily-tasks/blob/main/AGENTS.md) の「テストのコーディングルール」に従います。

### ArchitectureTest が検査するルール

[ArchitectureTest](https://github.com/KenichiroArai/gr002_daily-tasks/blob/main/src/test/java/kmg/gr/gr002/dailytasks/ArchitectureTest.java) は、次を満たさないと失敗します。層の意味は [01. 全体構成](01-architecture.md) です。

| ルール                         | 内容                                                                 |
| ------------------------------ | -------------------------------------------------------------------- |
| 層の依存                       | 参照はすぐ上の層だけ。飛び越しと、処理の流れと同じ向きの参照は禁止   |
| 依存の注入                     | 依存はコンストラクタで受け取る。コンストラクタが1つのときは `@Autowired` を付けない。2つ以上あるときは使うコンストラクタにだけ付ける。フィールドに `@Autowired` を付けない |
| `@Value`                       | 設定値を `@Value` で個別に取得しない                                 |
| `@ConfigurationProperties`     | presentation にだけ置く                                              |
| Jackson のアノテーション       | `repository/dto/json/` にだけ置く                                    |
| ロジックとサービス             | `logic/` は `service/` を参照しない                                  |

## メッセージ

文面はプロパティファイルに置き、キーは列挙型の `key` です。取得はインスタンス変数に注入した `DailyTasksMessageSource` を使います。`DailyTasksMessageSource` は kmg-fund の `KmgMessageSource` を継承し、追加ロジックは持ちません。ログは出力箇所ごとに固有のメッセージ ID とし、[共通のコーディングルール](../../coding-rules.md#ログ) の「ログ」に従います。

| ファイル                    | 列挙型              | 用途                         | 取得メソッド | 出力形式 |
| --------------------------- | ------------------- | ---------------------------- | ------------ | -------- |
| `messages-log.properties`   | `Gr002LogMsgTypes`  | ログ                         | `getLogMessage` | `[キー] 文面` |
| `messages.properties`       | `Gr002GenMsgTypes`  | 例外                         | `getExcMessage` | `[キー] 文面` |
| `messages.properties`       | `Gr002GenMsgTypes`  | 使い方など一般               | `getGenMessage` | 文面のみ |
| （バリデーション用の文面）  | `Gr002ValMsgTypes`  | 該当がなければ `NONE` だけ   | （同上の例外系） | `[キー] 文面` |

コードは `GR002_{GEN|LOG|VAL}` + 5桁です。種別はプレフィックスのみで表し、5桁は層構造です。

| 桁 | 意味 |
| -- | ---- |
| 1桁目 | 層。`0` 共通 / `1` domain / `2` infrastructure / `3` application / `4` presentation / `5` repository |
| 2桁目 | 層内パッケージ（例: presentation の `1` = command、application の `1` = service、repository の `1` = api.github） |
| 3〜5桁目 | 区分内の連番 `000`〜`999` |

番号の帰属は呼び出し側の実装パッケージです。空きにした旧番号は再利用しません。詳細は [gr002_daily-tasks の AGENTS.md](https://github.com/KenichiroArai/gr002_daily-tasks/blob/main/AGENTS.md) の「文字列（メッセージ）の管理」に従います。

引数なしで取得するため、メッセージ中の `{}` や `%s` / `%d` はそのまま返ります。ログは SLF4J の `{}`、例外は `String.format` の `%s` と `%d` で埋めます。取得した文面はいったん変数に入れてから渡します。解析の正規表現や日付書式はプロパティに置かず、domain の列挙型に置きます（[03. 収集処理](03-processing.md)）。

## コーディングルール

Java と TypeScript に共通する戻り値、処理コメント、分岐のルールは [共通のコーディングルール](../../coding-rules.md) を参照してください。メソッドの戻り値を判定するときだけ変数に入れ、変数そのものの比較は切り出しません。戻り値の変数 `result` はメソッドの最初の文で宣言し、戻り値以外の判定や別意味の中間値には使いません。初期値はガード節などで返すときだけ置き、読まれない初期化は書きません。可変長引数へ `int` を渡すときの `Integer.valueOf(int)` のような不要な明示ボックス化は避けます（詳細は [共通のコーディングルール](../../coding-rules.md) の「戻り値」と「ボックス化」）。メソッドの戻り値に続けてメソッドを呼ばず、いったん変数に受けます。`Stream`、Builder、`Comparator` の連結、文字列の `replaceAll().strip()` は定型として残します。実行時に文字列を結合するときは、kmg-core の `KmgString.concat` を使います。文字列リテラル同士、または文字列リテラルと定数の `+` は、コンパイル時に 1 つの文字列になるため、そのままにします。文字列の空・空白判定は `KmgString.isEmpty` / `isBlank` など、`List` / `Map` / 配列の空判定は `KmgListUtils` / `KmgMapUtils` / `KmgArrayUtils` を使います。日付・日時は書式が一致するときだけ `KmgLocalDateUtils` / `KmgLocalDateTimeUtils` を使い、JSON 境界の ISO `yyyy-MM-dd` と Instant の ISO-8601 は対応 API がないため JDK のままにします。
パッケージ構成、アクセス修飾子、`record` の禁止、文字列の結合、kmg-core / kmg-fund のユーティリティ、Javadoc、テストの書き方は [gr002_daily-tasks の AGENTS.md](https://github.com/KenichiroArai/gr002_daily-tasks/blob/main/AGENTS.md) に従います。

## ビルドと実行

gr002_daily-tasks でビルドし、daily-tasks-git をカレントにして実行します。

| コマンド | 内容                                                                 |
| -------- | -------------------------------------------------------------------- |
| `mvn test` | テストと JaCoCo の 100% チェック。レポートは `target/site/jacoco/index.html` |
| `mvn clean package` | テストと実行可能 jar の作成。成果物は `target/gr002_daily-tasks-0.1.0.jar` |
| `java -jar ../gr002_daily-tasks/target/gr002_daily-tasks-0.1.0.jar` | 差分モード。daily-tasks-git で実行する |
| 同上に `--full` | 全件モード                                                           |
| 同上に `--help` | 使い方の表示                                                         |

- 環境変数 `GITHUB_TOKEN` があると GitHub API の認証に使います。未設定でも公開リポジトリなら動きます
- 解析ルールや標準時間を変えたときは、全件モードで JSON を作り直します
- Eclipse の実行の構成は `launch/` にあります。収集の作業ディレクトリは兄弟の daily-tasks-git です。実行の構成のファイルにトークンは書きません

収集から GitHub Pages への公開までは、daily-tasks-git の [update-carryover.yml](https://github.com/KenichiroArai/daily-tasks-git/blob/main/.github/workflows/update-carryover.yml) が行います。流れは [全体構成](../../overview.md) と [フロントエンド設計書の 05. 品質・運用](../frontend/05-quality.md) を参照してください。

## 変更時の確認

収集ツールを変更した場合は、次を確認します。

1. `mvn test` で JaCoCo の 100% と `ArchitectureTest` が通ること
2. 解析ルールや JSON の形式を変えた場合は、全件モードで再生成できること
3. JSON の形式を変えた場合は、[Issue 解析の仕様](../../carryover-spec.md)、画面の zod スキーマ、[フロントエンド設計書の 02. データ設計](../frontend/02-data.md) を更新すること
4. 本設計書の関係する章を更新すること
