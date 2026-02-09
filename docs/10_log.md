# 10.トランザクションログ

## 10-1.アクティブログとアーカイブログ

### ロギング方式

Db2におけるトランザクションログのロギング方式には「循環ロギング」と「アーカイブロギング」の2種類がある。  

Db2インストール直後は循環ロギングの設定になっている。  

循環ロギングとは、アクティブログ(1次ログおよび2次ログ)ファイルをリング状に使用しもっとも古いログファイルを上書き利用していく方式である。  
長時間かかるトランザクションや大量のトランザクションが発生した場合に、使用可能なログファイルが枯渇して「ログフル」状態となって強制的にロールバックされてしまう。  
またこの方式ではログファイルが順次上書きされていくため、データベースのバックアップから復元した後にロールフォワードすることは出来ない。  
ロールフォワードを想定する必要のない開発環境等においては、ログファイルサイズが常に一定以下の容量に抑えられるというメリットがある。

アーカイブロギングとは、循環ロギングと異なるログファイルを上書き(再利用)しないロギング方式である。  
任意の時点にデータベースを復元することが出来る。  
バックアップから復元した後に任意の時点までロールフォワードすることが出来る。  
ログファイルは自動的に削除されないため、バックアップと合わせて適切に別ストレージへの移動や削除が必要となる。  

### （１）アクティブログ

未コミットなトランザクションログが記録されている「使用中」のログファイル。  
（厳密にはコミット済かつディスク書き込み待ちの情報も含まれる）

#### ～1次ログ～

あらかじめ確保されるログファイル。

#### ～2次ログ～

1次ログが不足した際に動的に追加される予備のログファイル。  
2次ログが設定上の最大数に到達すると「ログフル」状態となって強制的にロールバックされる。  

> [!NOTE]
> logprimary - 1 次ログ・ファイル数構成パラメーター  
> https://www.ibm.com/docs/ja/db2/12.1.x?topic=parameters-logprimary-number-primary-log-files  
> logsecond - 2 次ログ・ファイル数構成パラメーター  
> https://www.ibm.com/docs/ja/db2/12.1.x?topic=parameters-logsecond-number-secondary-log-files


### （２）アーカイブログ

コミットおよびディスクに書込み済のトランザクションログが記録されているログファイル。  

## 10-2.ログに関する設定の確認

### （１）データベース構成パラメータを確認する

```bash
db2 get db cfg for [データベース名] | grep LOG
```

### （２）データベース構成パラメータの読み方

1. ログファイルサイズ
    ```bash
    Log file size (4KB)                         (LOGFILSIZ) = 1024    # 4KB * 1024 = 4MB
    ```
2. 1次ログのファイル数
    ```bash
    Number of primary log files                (LOGPRIMARY) = 13      # 1次ログのファイル数（4MB * 13 = 52MB/最大）
    ```
3. 2次ログのファイル数
    ```bash
    Number of secondary log files               (LOGSECOND) = 12      # 2次ログのファイル数（4MB * 12 = 48MB/最大）
    ```


### （３）ログファイルの使用状況を確認する

1. 【db2pd】2次ログファイルの使用状況を確認する
    ```bash
    db2pd -db [データベース名] -logs        # SYSADM, SYSCTRL, SYSMAINT, SYSMONのいずれかの権限が必要
    ```
    ⇒ `Current Log Number`に「現在アクティブなログの数」が表示される。

2. 【SQL】ログ使用量と2次ログ使用状況を確認するクエリ
    ```sql
    SELECT
        MEMBER                                                                   AS "メンバー番号",
        DECIMAL(TOTAL_LOG_USED / 1024.0 / 1024.0, 18, 2)                         AS "現在使用中ログ容量(MB)",
        DECIMAL(TOTAL_LOG_AVAILABLE / 1024.0 / 1024.0, 18, 2)                    AS "ログ総容量(MB)",
        DECIMAL((TOTAL_LOG_AVAILABLE - TOTAL_LOG_USED) / 1024.0 / 1024.0, 18, 2) AS "ログ残容量(MB)",
        DECIMAL(100.0 * TOTAL_LOG_USED / NULLIF(TOTAL_LOG_AVAILABLE, 0), 6, 2)   AS "ログ使用率(%)",
        DECIMAL(SEC_LOG_USED_TOP / 1024.0 / 1024.0, 18, 2)                       AS "2次ログ最大使用量(MB)",
        DECIMAL(TOT_LOG_USED_TOP / 1024.0 / 1024.0, 18, 2)                       AS "ログ最大使用量(履歴ピーク)(MB)",
        SEC_LOGS_ALLOCATED                                                       AS "現在割り当て中の2次ログ数"
    FROM TABLE(SYSPROC.MON_GET_TRANSACTION_LOG(-2)) AS T
    ORDER BY MEMBER;
    ```
    > [!NOTE]
    > SQLファイルに保存しておきCLP(db2コマンド)で実行するように準備しておくことを推奨。  
    > db2pdの手順と異なり特別な権限は不要。  


## 10-3.ログに関する設定の変更

### （１）循環ロギングからアーカイブロギングに変更

1. 事前にバックアップを取得
    ```bash
    db2 backup db [データベース名] to [バックアップ先パス]
    ```
2. 設定変更
    ```bash
    db2 update db cfg for [データベース名] using NEWLOGPATH [アクティブログ出力ディレクトリ]
    db2 update db cfg for [データベース名] using LOGARCHMETH1 DISK:[アーカイブログ出力ディレクトリ]
    ```
    > [!NOTE]
    > ログ出力ディレクトリのパス設計例。ログだからと言って/var/log配下にするのは駄目だと思う。
    > * アクティブログ : /data/db2/actlog
    > * アーカイブログ : /data/db2/arclog
3. 再起動
    ```bash
    db2stop
    db2start
    ```
4. 事後のバックアップを取得（ロールフォワード用）
    ```bash
    db2 backup db [データベース名] to [バックアップ先パス]
    ```
    > [!NOTE]
    > バックアップの進行状況は`db2 list utilities show detail`で確認可能。

### （２）2次ログファイル数の変更

1. 変更前の設定確認
    ```bash
    db2 get db cfg for [データベース名] | grep LOGSECOND
    ````
2. 設定変更
    ```bash
    db2 update db cfg for [データベース名] using LOGSECOND [変更値] IMMEDIATE
    ```

3. 変更後の設定確認
    ```bash
    db2 get db cfg for [データベース名] | grep LOGSECOND
    ````


