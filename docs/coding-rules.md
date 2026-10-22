# Java と TypeScript に共通するコーディングルール

収集ツール（Java、[gr002_daily-tasks](https://github.com/KenichiroArai/gr002_daily-tasks)）と画面（TypeScript、[mk002_daily-tasks_web-next](https://github.com/KenichiroArai/mk002_daily-tasks_web-next)）に共通するルールです。
言語ごとの詳細なルールは、それぞれのリポジトリの `AGENTS.md` に記載します。

## 戻り値

- 戻り値は変数 `result` で定義する
- 戻り値の変数は先頭で宣言する
- return 文は `return result;` に統一する
- TypeScript で対象外とするもの（式だけを返す短いアロー関数、React コンポーネントの JSX など）は mk002_daily-tasks_web-next の `AGENTS.md` を参照する

## 早期リターン

- 早期リターン（ガード節）を使用し、不要なネストを避ける
- 条件が満たされない場合は早期に `return` する。ガード節では `result` の初期値を返す
- if-else の代わりにガード節を使い、インデントの深さを最小限に抑える

```java
public boolean someMethod(String input) {
    boolean result = false;  // 先頭で戻り値変数を宣言

    // 早期リターン（ガード節）
    if (input == null) {
        return result;
    }

    // メインの処理
    result = true;

    return result;  // 統一された形式で return
}
```

```ts
export function statValue(stat: Stat | undefined, metric: Metric): number {
  let result = 0;
  if (!stat) {
    return result;
  }
  result = metric === "count" ? stat.count : stat.minutes;
  return result;
}
```

## 構成

- 機能単位で構成する（Java は機能パッケージ + 5 層、TypeScript は `src/features/<機能名>/`）
- 依存の向きを一方向に限定し、ツール（Java は ArchUnit、TypeScript は ESLint）で検査する

## 役割

- 1つの型（クラス、モジュール、関数のまとまり）には、1つの役割の機能だけを置く
- 同じモデルや同じ設定を受け取ることではまとめない
- 分ける単位は、呼び出し側が別々に頼む機能のまとまりとする
- 1つの操作だけを切り出したり、同じ役割の読み書きを別の型に分けたりしない
- 一連の流れを行うサービスやコマンドは、その流れが1つの役割なら分けない
- 取得・分析・保存のように役割が複数あるサービスは、統合サービスを残して役割ごとのサービスに分ける。呼び出し元が呼ぶのは役割までとし、役割の中の段階は統合サービスが順に呼ぶ
- 保存だけのサービスは作らない。解析結果の保存は収集のメソッド、集計の保存は集計のメソッドとする

## 対応

キーや値に業務上の意味があり、メソッドやクラスの外へ渡す対応は、名前のある型にします。

- Java はモデルまたは DTO のクラスにします。`Map` を公開の引数や戻り値にしません
- TypeScript は型エイリアスにします。匿名の `Map` や `Record` を公開の引数や戻り値にしません。画面はクラスにしません
- メソッド内の一時集計と、クラスやモジュール内の索引は、`Map` や `Record` のままでかまいません。使う理由をコメントまたは Javadoc に書くことを推奨します
- 残してよい例: 列挙型の `VALUES_MAP`（定数名から定数を引く索引）、Jackson が動的キーの JSON を読むときのメソッド内の `Map`、集計メソッド内の件数カウント、画面のラベル定数
- 中身の `Map` や `Record` を型の表現として残すときは、その理由を書きます。JSON の動的キーがこれに当たります
- 言語ごとの例は、各リポジトリの `AGENTS.md` を参照します

## データ

- daily-tasks-git の `docs/data/` の JSON を作成・変更するのは収集ツールだけとし、画面は読み取りだけにする
- `docs/data/` の JSON を手で編集しない
- シークレット（`GITHUB_TOKEN` など）をコード・ログ・JSON に出さない

## Javadoc（Java）

- `{@inheritDoc}` は使わない。オーバーライドやインタフェースの実装でも、そのメソッドに説明と必須タグを書く
- タグや書き方の詳細は [gr002_daily-tasks の AGENTS.md](https://github.com/KenichiroArai/gr002_daily-tasks/blob/main/AGENTS.md) の「Javadoc」に従う

## テスト

- テストで GitHub API に実通信しない
- 計算ロジックは純粋な関数・メソッドに切り出し、単体テストを書く
