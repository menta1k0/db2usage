# 7.統計情報(RUNSTATS)

## 統計情報の更新

統計情報を最新に保つためRUNSTATSコマンドを実行する（＝統計情報を更新する）。  
表の再編成を実施した後は必ず統計情報を更新すること。また必要に応じてパッケージの再バインドも行うこと。

### 基本

```sql
RUNSTATS ON TABLE [スキーマ名].[テーブル名] ON ALL COLUMNS WITH DISTRIBUTION AND INDEXES ALL ALLOW WRITE ACCESS
```

### サンプリング率指定
```sql
RUNSTATS ON TABLE [スキーマ名].[テーブル名] ON ALL COLUMNS WITH DISTRIBUTION AND INDEXES ALL ALLOW WRITE ACCESS TABLESAMPLE SYSTEM (1)
```
> [!NOTE]
>数千万レコード以上あるテーブルの場合、サンプリング率1％でもかなりの精度の統計情報が取得できる。

### RUNSTATSコマンドの進捗状況確認
```sql
list utilities show detail
```

## 統計情報の確認

### SYSCAT.TABLES
```sql
SELECT TABSCHEMA, TABNAME, CARD, STATS_TIME FROM SYSCAT.TABLES WHERE [抽出条件]
```
> [!NOTE]
> 項目の概要
> |項目名|説明|
> |:--|:--|
> |TABSCHEMA|スキーマ名|
> |TABNAME|テーブル名|
> |CARD|表内の行の総数。統計が収集されていない場合は -1。サンプル率を指定してRANSTATSした場合は推定行数。|
> |STATS_TIME|統計情報更新日時|