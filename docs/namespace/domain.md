# Domain の作法

| 項目 | 内容 |
|---|---|
| ID | namespace-4 |
| 分類 | namespace |
| 関連 | namespace-1(総覧) / namespace-5(Entity のコード値カラム) / namespace-2(業務定数との線引き) / data-1(アクセサ) / test-1(純粋ロジックのテスト) |

## 目的

業務知識(コード値・純粋ロジック・桁数)を `Domain` に集約し、I/O やフレームワークから切り離す。

- コード値の定義と判定が一箇所になり、マジックナンバーが消える
- 純粋ロジックは引数と戻り値だけで完結し、単体テストが容易になる
- **Domain は他レイヤに依存しない**。参照方向は常に「他レイヤ → Domain」の一方向

## 標準形

### フォルダ構成

```
Domain/
├─ Code/       # コード値定数(static class + const)
│  ├─ AlertLevel.cs
│  └─ UserType.cs
├─ Logic/      # I/O 非依存の純粋ロジック
│  └─ PriceLogic.cs
└─ Length.cs   # 桁数定数の集約
```

### Domain/Code — コード値は enum ではなく static class + const

DB の数値/文字列カラムと直結する値は **enum ではなく `static class` + `const`** で定義し、判定メソッドを同居させる。

```csharp
namespace Template.Domain.Code;

public static class AlertLevel
{
    public const sbyte None = 0;
    public const sbyte Warning = 1;
    public const sbyte Error = 2;

    public static bool IsAlert(sbyte value) => value > None;
}
```

- Entity のプロパティはカラム型そのまま(`sbyte` / `string`)。変換・キャストが発生しない
- 判定ロジックが値定義と同居し、利用側は `AlertLevel.IsAlert(entity.Level)` と書ける

enum を使わない理由:

| 観点 | enum | static class + const |
|---|---|---|
| DB 値との対応 | キャスト・変換が必要 | カラム型と 1:1、変換不要 |
| 未定義値の扱い | 未定義値もキャストで生成でき、正当性検証が別途必要 | 値はただの数値/文字列。定義済み判定もメソッドで表現 |
| 判定ロジック | 拡張メソッドを別クラスに書くことになる | 定数と同じクラスに同居 |
| ビット演算・比較 | キャストの連続になりがち | 素の演算子でそのまま書ける |

文字列コードも同形で定義する。

```csharp
namespace Template.Domain.Code;

public static class UserType
{
    public const string Admin = "A";
    public const string Member = "M";

    public static bool IsAdmin(string value) => value == Admin;
}
```

### enum で良いケース

**アプリ内で閉じる状態**は enum で表現してよい。

- DB・API 境界に出ない画面状態・接続状態などのステート表現
- フレームワークが enum を要求する箇所

境界を跨ぐ(DB に保存する・API で送る)値になった時点で `Domain/Code` の const へ移す。

### Domain/Logic — 純粋ロジック

I/O 非依存(DB・通信・時刻取得をしない)の業務計算をクラスとして置く。

```csharp
namespace Template.Domain.Logic;

public static class PriceLogic
{
    public static int CalcTotal(int unitPrice, int quantity, decimal taxRate) =>
        (int)Math.Floor(unitPrice * quantity * (1 + taxRate));
}
```

入出力が引数と戻り値だけなので、テストは AAA の数行で書ける(test-1)。現在時刻のような環境依存値は引数で受け、取得は上位に任せる(`Providers` → namespace-1)。

### Length — 桁数定数の集約

入力制限・検証・DB 定義で共有する桁数を `Length` クラスに集約する。

```csharp
namespace Template.Domain;

public static class Length
{
    public const int UserId = 8;
    public const int UserName = 40;
    public const int Password = 256;
}
```

## 配置ルール

| 対象 | 場所 |
|---|---|
| コード値定数 | `Domain/Code/`(値種別毎に1クラス) |
| 純粋ロジック | `Domain/Logic/` |
| 桁数定数 | `Domain/Length.cs` |
| アプリ内で閉じる enum | 利用箇所のレイヤ(`State` / `Models` 等) |
| Core 分離構成(solution-1) | `Domain` は Core プロジェクト側 |

依存方向: `Domain` は BCL 以外を参照しない。`Models` / `Services` / `Usecase` / `Endpoints` から参照される側であり、逆参照しない。

## バリエーションと使い分け

- 判定が複雑化したら `Code` の static メソッドから `Logic` のクラスへ昇格させる
- 業務定数の線引き: 業務仕様に由来する値(コード値・桁数)は `Domain`、アプリ運用のポリシー値(ページサイズ・件数上限)は `Application` の `LimitPolicy`(namespace-2)
- クライアント側でも同じ作法で `Domain` を持つ(namespace-7)

## アンチパターン

- コード値を enum で定義して DB 値とキャスト変換する — const で 1:1 に保つ
- マジックナンバーの直書き(`if (entity.Level > 0)`)— 必ず `AlertLevel.IsAlert` のような判定メソッドを通す
- `Domain` から Service・Accessor・設定クラスを参照する — 必要な値はすべて引数で受ける
- 桁数リテラルを検証属性・SQL・画面に散らす — `Length` 参照に統一する
- `DateTime.Now` を `Logic` 内で呼ぶ — 時刻は引数で受ける
