# 8.実行計画(アクセスプラン)

## 初回実行時のみ

EXPLAIN表を作成しておく必要がある。
```sh
db2 -tvf /home/db2inst1/sqllib/misc/EXPLAIN.DDL
```
> [!NOTE]
> 上記のDDLファイルのパスはインスタンスオーナーがdb2inst1の場合の例。  
> インスタンスオーナーの`$HOME/sqllib/misc/EXPLAIN.DDL`となる。

## 実行計画をファイルに出力する

1. モード変更
    ```sh
    db2 "set current explain mode explain"
    ```
    
    上記の設定の後にSQLを実行しても実際にデータアクセスは行われない。  
    `explain mode yes`にすると実際にデータアクセスも行われるようになる（こちらを利用する場面はほぼ無い）。
2. 実行計画を取得したいSQLを実行
    ```sh
    db2 -tvf [SQLファイル]
    ```
3. モード変更
    ```sh
    db2 "set current explain mode no"
    ```
4. フォーマット出力
    ```sh
    db2exfmt -1 -d [データベース名] -o [出力ファイル名]
    ```