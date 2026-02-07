# 3.コマンドラインプロセッサ(CLP)の使用方法

## 3-1. SQLの実行

### データベースへの接続

#### 基本形
```sh
db2 connect to [データベース名]
```

#### ユーザーを指定して接続する場合（パスワードは対話的に入力）
```sh
db2 connect to [データベース] user [接続ユーザー]
```

#### 非推奨
```sh
db2 connect to [データベース] user [接続ユーザー] using [パスワード]
```

### スキーマの変更

#### デフォルトスキーマは接続ユーザー名と同名となるので、カレントスキーマを目的のスキーマに変更する
```sh
db2 set schema [スキーマ名]
```

### データベースからの切断

#### クライアント接続解除
```sh
db2 connect reset
```

#### クライアント接続解除＋バックグラウンドプロセス停止
```sh
db2 terminate
```
> [!NOTE]
>全てのバックグラウンドプロセス(db2bp)が停止するわけではない。

### SQLの実行

1. SQLを直接指定
    ```sh
    db2 "SELECT * FROM [テーブル名] WHERE [抽出条件]"
    ```
    > [!NOTE]
    > SQLを直接指定する場合はステートメント終了文字(セミコロン)は不要。  
    > セミコロンを付けて複数のステートメントを1行で記述することは不可。

2. SQLファイルを指定
    ```sh
    db2 -tvf [SQLファイルのパス]
    ```
    > [!NOTE]
    > Db2のプロセスがSQLファイルにアクセス可能になっている必要がある。  
    > 「SQLファイルが無い」という様なエラーになる場合は、ディレクトリやファイルの権限を確認してみること。

    > [!NOTE]
    > オプションの解説：  
    > t ...  ステートメント終了文字としてセミコロン (;) を使用する  
    > v ...  コマンド・テキストを標準出力にエコー出力する  
    > f ...  標準入力からではなくファイルからコマンド入力を読み取る  

## 3-2. PL/SQLの実行

PL/SQLを実行するためにはOracle互換モードが有効になっている必要がある。
> [!NOTE]
> Oracle互換モードの設定方法については下記ページを参照。  
> [1.環境構築](1_env.md#1-4-データベースの作成) の「1. Oracle互換モードを使用する場合（使用しない場合はSKIP）」

1. サンプルPL/SQL(hello_world.sql)
    ```sql:hello_world.sql
    DECLARE
        V_MSG   VARCHAR(500)  DEFAULT '初期メッセージ';
    BEGIN
        DBMS_OUTPUT.PUT_LINE(V_MSG);
        V_MSG := 'Hello World!';
        DBMS_OUTPUT.PUT_LINE(V_MSG);
    END;
    /
    ```
    実行結果：
    >DB20000I  The SQL command completed successfully.  
    >初期メッセージ  
    >Hello World!  

2. サンプルPL/SQLを実行する
    ```sh
    db2 "set serveroutput on"
    db2 -td/ -f hello_world.sql
    ```
    > [!NOTE]
    >このサンプルでは標準出力機能(DBMS_OUTPUT.PUT_LINE)を利用しているので「set serveroutput on」が必要。  

    > [!NOTE]
    > ステートメント区切り文字にスラッシュではなくアットマークを使用する場合は、  
    > 「db2 -td@ -f hello_world.sql」という様に指定すればよい。