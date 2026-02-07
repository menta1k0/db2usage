# 7.統計情報(RUNSTATS)

## 統計情報の更新

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
>TABSCHEMA … スキーマ名  
>TABNAME … テーブル名  
>CARD … 件数(カーディナリティ)  
>STATS_TIME … 統計情報更新日時