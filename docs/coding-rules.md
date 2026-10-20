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

## データ

- daily-tasks-git の `docs/data/` の JSON を作成・変更するのは収集ツールだけとし、画面は読み取りだけにする
- `docs/data/` の JSON を手で編集しない
- シークレット（`GITHUB_TOKEN` など）をコード・ログ・JSON に出さない

## テスト

- テストで GitHub API に実通信しない
- 計算ロジックは純粋な関数・メソッドに切り出し、単体テストを書く
