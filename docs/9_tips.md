# 9.知っていると便利なトピック

## 9-1.テーブル定義

### 9-1-1.カラム名・型・桁・NULL可否の確認

* 基本的なテーブル定義の確認
    ```sh
    db2 describe table [テーブル名]
    ```

* 詳細なテーブル定義の確認
    ```sql
    SELECT * FROM SYSCAT.COLUMNS WHERE TABSCHEMA = [スキーマ名] AND TABNAME = [テーブル名]
    ```

### 9-1-2.統計情報(統計情報更新日時・レコード件数)

* [統計情報更新方法](7_runstats.md)

* 統計情報更新日時・レコード件数の確認
    ```sql
    SELECT CARD, STATS_TIME FROM SYSCAT.TABLES WHERE TABSCHEMA = [スキーマ名] AND TABNAME = [テーブル名]
    ```
    > [!NOTE]
    > 項目の概要
    > |項目名|説明|
    > |:--|:--|
    > |TABSCHEMA|スキーマ名|
    > |TABNAME|テーブル名|
    > |CARD|表内の行の総数。統計が収集されていない場合は -1。サンプル率を指定してRANSTATSした場合は推定行数。|
    > |STATS_TIME|統計情報更新日時|

## 9-2.インデックス定義

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
SELECT TABNAME,INDNAME,COLNAMES FROM SYSCAT.INDEXES WHERE TABSCHEMA = [スキーマ名] AND TABNAME = [テーブル名] AND UNIQUERULE = 'D'
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

>[!NOTE]
> DBMSごとの文字数を返す関数は下表のとおり。
> |DBMS|文字数を返す関数|
> |:--|:--|
> |Db2|CHAR_LENGTH|
> |MySQL / MariaDB|CHAR_LENGTH|
> |Oracle|LENGTH|
> |PostgreSQL|LENGTH|
> |SQL Server|LEN|

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
    TABSCHEMA, TABNAME
```

## 9-6.クエリの調査

### 9-6-1.実行中クエリの特定

```sql
-- MON_CURRENT_SQL管理ビュー を利用して現在実行中のクエリを取得する
SELECT
    CURRENT_TIMESTAMP           AS TIMESTAMP,   -- 現在日時
    APPLICATION_HANDLE,                         -- db2diag.log等で詳細調査時に重要な番号
    APPLICATION_NAME,                           -- クライアント情報等が確認できる
    APPLICATION_ID,                             -- ステートメントを実行しているIPアドレスも確認できる
    CURRENT CLIENT_APPLNAME,                    -- (特殊レジスタ)
    ACTIVITY_TYPE,                              -- LOAD, READ_DML, WRITE_DML等
    ACTIVITY_STATE,                             -- IDLE, EXECUTING等
    ELAPSED_TIME_SEC,                           -- 経過時間(秒)
    TOTAL_CPU_TIME,                             -- 総CPU時間
    ROWS_READ,                                  -- 読み取り行数
    ROWS_RETURNED,                              -- 戻り行数
    QUERY_COST_ESTIMATE,                        -- 照会コスト見積
    SUBSTR(STMT_TEXT,1,500)      AS STMT_TEXT   -- ステートメント
FROM
    SYSIBMADM.MON_CURRENT_SQL
```

### 9-6-2.実行時間が長いクエリの特定

```sql
-- MON_GET_PKG_CACHE_STMT 表関数から、実行時間合計のトップ20を取得
SELECT
    -- 実行回数と時間
    NUM_EXECUTIONS,                                                             -- 実行回数
    CAST(TOTAL_CPU_TIME / 1000 AS DEC(15,2)) AS TOTAL_CPU_MS,                   -- CPU時間合計(ミリ秒)
    CAST(STMT_EXEC_TIME / 1000 AS DEC(15,2)) AS TOTAL_EXEC_MS,                  -- 実行時間合計(ミリ秒)
    CAST(CASE
        WHEN NUM_EXECUTIONS > 0 THEN (FLOAT(STMT_EXEC_TIME) / FLOAT(NUM_EXECUTIONS)) / 1000
        ELSE 0
    END AS DEC(15,2)) AS AVG_EXEC_MS,                                           -- 平均実行時間(ミリ秒)

    -- 読み取り（論理読取に対して物理読取が小さいほど、バッファプール命中が高い“傾向”）
    (POOL_DATA_L_READS + POOL_INDEX_L_READS) AS TOTAL_LOGICAL_READS,            -- 合計論理読み取り
    (POOL_DATA_P_READS + POOL_INDEX_P_READS) AS TOTAL_PHYSICAL_READS,           -- 合計物理読み取り（バッファプールへの物理読込）

    -- 参考：物理読取率（0に近いほどバッファプール命中が高い傾向）
    CAST(CASE
        WHEN (POOL_DATA_L_READS + POOL_INDEX_L_READS) > 0
        THEN FLOAT(POOL_DATA_P_READS + POOL_INDEX_P_READS) / FLOAT(POOL_DATA_L_READS + POOL_INDEX_L_READS)
        ELSE NULL
    END AS DEC(9,6)) AS PHYSICAL_READ_RATIO,                                    -- 物理読取率

    -- 待機時間割合
    CAST(CASE
        WHEN STMT_EXEC_TIME > 0 THEN (FLOAT(TOTAL_ACT_WAIT_TIME) / FLOAT(STMT_EXEC_TIME)) * 100
        ELSE 0
    END AS DEC(5,2)) AS WAIT_PCT,                                               -- 実行時間に占める待機時間割合(%)

    -- SQL文 (先頭 200 文字)
    SUBSTR(STMT_TEXT, 1, 200) AS STMT_TEXT                                      -- ステートメント
FROM
    TABLE(MON_GET_PKG_CACHE_STMT(NULL, NULL, NULL, -1))
ORDER BY
    STMT_EXEC_TIME DESC
FETCH FIRST 20 ROWS ONLY
```

## 9-7.ストアドファンクション・ストアドプロシージャ

### 9-7-1.「CURRENT_SCHEMA」を指定してもエラーになる理由

#### 事象
- スキーマ名を省略してファンクションやプロシージャを作成すると、CURRENT_SCHEMAではなくユーザー自身のスキーマに作成される
- スキーマ名を明示してファンクションやプロシージャを作成した後に、スキーマ名を省略して呼び出すと発見できずにエラーが発生する（SQL0440N, SQLSTATE=42884）

#### 理由  
- CURRENT_SCHEMAが効くのはテーブルのみ
- ファンクション・プロシージャのスキーマ名は「CURRENT PATH 特殊レジスター」の設定に従い暗黙的に解決される

#### 解決方法
- 「CURRENT PATH 特殊レジスター」にユーザー定義のファンクションやプロシージャを作成したスキーマ名を指定する（複数指定時はカンマ区切りする）
    - SQLで設定する場合
    カレントスキーマを指定するのと同じ要領で以下の通りスキーマ名を指定する。
    ```sql
    SET PATH = [スキーマ名]
    ```
    - JDBCの接続URLに設定する場合  
    currentFunctionPathというパラメタにスキーマ名を指定する。
    ```
    jdbc:db2://${hostName}:${portNo}/${dbName}:currentSchema=${schemaName};currentFunctionPath=${schemaName};
    ```

#### 参考情報

- 「CURRENT PATH 特殊レジスター」には暗黙的に常にSYSIBM, SYSFUN, SYSPROC, SYSIBMADMが設定される。  
```SET PATH = [スキーマ名]```に1つのスキーマしか指定しなかったとしてもこの4つのスキーマは暗黙的に参照される。  
参考：https://www.ibm.com/docs/ja/db2-for-zos/12.0.0?topic=statements-set-path