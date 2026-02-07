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

* [統計情報更新方法](../docs/7_runstats.md)

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

## 9-5.ADMINTABINFO管理ビュー

### 9-5-1.テーブルの状態確認

REORGやLOADによって意図せぬPENDING状態に陥ったと思われる場合は下記のSQLを実行して状態を確認する。
```sql
SELECT
  VARCHAR(TABSCHEMA, 10)  AS TABSCHEMA,
  VARCHAR(TABNAME, 50)    AS TABNAME,
  TABTYPE,                  -- T:通常表、S:MQT、H:階層表
  AVAILABLE,                -- Y:使用可能、N:使用不能
  -- パーティション関連
  DBPARTITIONNUM,
  DATA_PARTITION_ID,
  -- 再編成関連
  REORG_PENDING,            -- Y:オフライン再編成が必要、N:左記以外
  INPLACE_REORG_STATUS,     -- NULL、EXECUTING、PAUSED:一時停止(RESUME可能)、ABORTED:RESUME不可(STOP必須)
  INDEXES_REQUIRE_REBUILD,  -- Y:インデックスのリビルドが必要な場合、N:左記以外の場合
  -- LOAD関連
  LOAD_STATUS,              -- NULL、IN_PROGRESS、PENDING
  READ_ACCESS_ONLY,         -- Y:読み取り専用状態、N:左記以外
  NO_LOAD_RESTART           -- Y:部分的なLOADとなってしまっておりLOADが再開できない、N:左記以外
FROM
  SYSIBMADM.ADMINTABINFO
WHERE
  TABSCHEMA = [スキーマ名] AND TABNAME = [テーブル名]
```

### 9-5-2.テーブルのサイズ(KB)

パーティション表の場合は全体の合計サイズになる。パーティション単位のサイズが知りたい場合はGROUP BYを外す。  
なおこの方法で確認できるのは物理サイズ(PhysicalSize)であり、再編成(REORG)が長期実行されていないテーブルでは実際のデータ量より過剰になる。  
※INSPECTコマンドを使用すれば精緻なデータサイズを確認することが可能だが、性能リスクが高いため利用には注意が必要。
```sql
SELECT
    VARCHAR(TABSCHEMA, 10)      AS TABSCHEMA,
    VARCHAR(TABNAME, 50)        AS TABNAME,
    SUM(DATA_OBJECT_P_SIZE)     AS DATA_OBJECT_KB,
    SUM(INDEX_OBJECT_P_SIZE)    AS INDEX_OBJECT_KB,
    SUM(LONG_OBJECT_P_SIZE)     AS LONG_OBJECT_KB,
    SUM(LOB_OBJECT_P_SIZE)      AS LOB_OBJECT_KB,
    SUM(XML_OBJECT_P_SIZE)      AS XML_OBJECT_KB
FROM
    SYSIBMADM.ADMINTABINFO
WHERE
    TABSCHEMA = [スキーマ名]
GROUP BY
    TABNAME
```