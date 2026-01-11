# 9.知っていると便利なトピック

## 9-1.テーブル定義(SYSCAT.TABLES)

### 9-1-1.カラム名・型・桁・NULL可否の確認

* 基本的なテーブル定義の確認
    ```sh
    db2 describe table [テーブル名]
    ```

* 詳細なテーブル定義の確認
    ```sql
    SELECT * FROM SYSCAT.TABLES WHERE TABSCHEMA = [スキーマ名] AND TABNAME = [テーブル名]
    ```

### 9-1-2.統計情報(統計情報更新日時・レコード件数)

* [統計情報更新方法](../docs/8_plan.md)

* 統計情報更新日時・レコード件数の確認
    ```sql
    SELECT CARD, STATS_TIME FROM SYSCAT.TABLES WHERE TABSCHEMA = [スキーマ名] AND TABNAME = [テーブル名]
    ```
    > [!NOTE]
    >TABSCHEMA … スキーマ名  
    >TABNAME … テーブル名  
    >CARD … 件数(カーディナリティ)  
    >STATS_TIME … 統計情報更新日時

## 9-2.インデックス定義(SYSCAT.INDEXES)

### 9-2-1.主キーの確認

```sql
SELECT TABNAME,INDNAME,COLNAMES FROM SYSCAT.INDEXES WHERE TABSCHEMA = [スキーマ名] AND TABNAME = [テーブル名] AND UNIQUERULE = 'P'
```

### 9-2-2.ユニークインデックスの確認

```sql
SELECT TABNAME,INDNAME,COLNAMES FROM SYSCAT.INDEXES WHERE TABSCHEMA = [スキーマ名] AND TABNAME = [テーブル名] AND UNIQUERULE = 'U'
```

### 9-2-3.インデックスの確認

```sql
SELECT TABNAME,INDNAME,COLNAMES FROM SYSCAT.INDEXES WHERE TABSCHEMA = [スキーマ名] AND TABNAME = [テーブル名] AND UNIQUERULE = ''
```

## 9-3.Db2固有の関数・機能

### 9-3-1.ダミー表

OracleのDUAL表に相当する `SYSIBM.SYSDUMMY1` というテーブルがある。
```sql
SELECT CURRENT_TIMESTAMP FROM SYSIBM.SYSDUMMY1
```
>[!NOTE]
>Oracle互換モード=ORAのデータベースの場合はDUAL表を利用できる。

## 9-4.よく間違える関数

### 9-4-1.文字列の桁数(CHAR_LENGTH)

「文字列が5桁以上のものを抽出」したい場合に `WHERE LENGTH(文字列) >= 5` と指定してしまうと、
5バイト以上のものが抽出されてしまい、日本語等のマルチバイトなデータを正しく扱えない。  
桁数を条件指定する場合は `CHAR_LENGTH` を使用する。

`CHAR_LENGTH` はOracle、SQL Serverでは利用できない。
MySQLとPostgreSQLでは利用できる（CHARACTER_LENGTHも同義）。



