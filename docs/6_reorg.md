# 6.再編成(REORG)

## 再編成の種類

表(テーブル)を再編成する方法は4つある。  
ここではオンライン再編成(インプレース再編成)について解説する。

4つの方法についてはIBM公式サイトを参照すること。  
https://www.ibm.com/docs/ja/db2/12.1.x?topic=reorganization-choosing-table-method

## インプレース再編成の特徴

* オフライン再編成と異なりREORG PENDINGになることは無く、オンラインのまま再編成することができるが、  
その分トランザクションログを使用する（再編成中の異動を退避しておく必要があるため）
* 表再編成が完了してもインデックス再編成は実施されないため、別途インデックス再編成を実行する必要がある  
（オフライン再編成の場合は、表再編成に続いてインデックス再編成が自動実行される）

## 再編成関連のコマンド

### 再編成要否の確認

#### 既存の統計情報を元に判定

```sql
REORGCHK CURRENT STATISTICS ON TABLE [スキーマ名].[テーブル名]
```

#### 最新の統計情報を元に判定（統計情報が更新されるため注意）

```sql
REORGCHK UPDATE STATISTICS ON TABLE [スキーマ名].[テーブル名]
```
> [!NOTE]
>統計情報の更新方法を細かく指定できないため、”普段通りの統計情報の更新”を行ってから「既存の統計情報を元に判定」を実施した方が良い。

### テーブルのインプレース再編成

* 開始
    ```sql
    REORG TABLE [スキーマ名].[テーブル名] INPLACE ALLOW WRITE ACCESS
    ```
* 中断
    ```sql
    REORG TABLE [スキーマ名].[テーブル名] INPLACE PAUSE
    ```
* 中断からの再開
    ```sql
    REORG TABLE [スキーマ名].[テーブル名] INPLACE RESUME
    ```
* 停止（再開不可。次回実行時は最初から開始）
    ```sql
    REORG TABLE [スキーマ名].[テーブル名] INPLACE STOP
    ```

> [!NOTE]
>パーティションテーブルの場合はテーブル名に加えてパーティションも指定する必要がある。
>詳細はIBM公式サイトをよく確認すること。
>https://www.ibm.com/docs/ja/db2/12.1.x?topic=commands-reorg-table

### インデックスの再編成

```sql
REORG INDEXES ALL FOR TABLE [スキーマ名].[テーブル名] ALLOW WRITE ACCESS
```
> [!NOTE]
>パーティションテーブルのインデックスの場合は別途考慮すべき点があるので、IBM公式サイトをよく確認すること。  
> https://www.ibm.com/docs/ja/db2/12.1.x?topic=commands-reorg-indexindexes