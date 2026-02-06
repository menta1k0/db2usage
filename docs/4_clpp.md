# 4.コマンドラインプロセッサPlus(CLPPlus)の使用方法

リモートデータベースをカタログせずに接続できるため便利であるが、CLPと比べると操作性に若干難がある。  
一般的なSQLの操作（SQLファイルの実行を含む）をするケースではCLPで必要十分なことが多い。  
PL/SQLのバインド変数を利用して戻り値をシェルに返したいケースの様な、CLPでは機能的に実現できない場合にCLPPlusを利用することになる。

## 4-1.SQLの実行

### データベースへの接続

1. CLPPlusの起動と共に接続する場合
    ```bash
    clpplus -nw [ユーザー名]/[パスワード]@[ホスト名]:[ポート番号]/[データベース名]
    ```

    > [!NOTE]
    >パスワードの記載を省略した場合は、対話モードでパスワードの入力が求められる。

2. 後から接続する場合は、clpplusコマンドを単独実行した後にCONNECTする。
    ```bash
    clpplus -nw
    >SQL CONNECT [ユーザー名]/[パスワード]@[ホスト名]:[ポート番号]/[データベース名];  
    ```

### スキーマの変更

#### デフォルトスキーマは接続ユーザー名と同名となるので、カレントスキーマを目的のスキーマに変更する
```sh
SQL> SET CURRENT_SCHEMA = [スキーマ名];
```

### データベースからの切断

```bash
SQL> DISCONNECT;
```

### CLPPlusを終了

```bash
SQL> EXIT;
```

### SQLの実行

1. SQLを直接指定
    ```bash
    SQL> "SELECT * FROM [テーブル名] WHERE [抽出条件]";
    ```

2. SQLファイルを指定
    ```bash
    SQL> @[SQLファイルのパス]
    ```

3. CLPPlusの起動・データベース接続・SQLファイルの実行を1行で（パスワードを省略すると対話モードで入力要求される）
    ```bash
    clpplus -nw [ユーザー名]/[パスワード]@[ホスト名]:[ポート番号]/[データベース名] @[SQLファイルパス]
    ```
    > [!NOTE]
    >このやり方の場合はSQLファイルの内容を実行した後に対話モードが継続する。DISCONNECTとEXITの手打ちが必要になる。

## 4-2.PL/SQLの実行

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

2. サンプルPL/SQLを実行する（対話入力）
    ```bash
    SQL> SET SERVEROUTPUT ON;
    SQL> @hello_world.sql
    SQL> DISCONNECT;
    SQL> EXIT;
    ```

3. サンプルPL/SQLをヒアドキュメントを利用してシェルから自動実行する（対話入力なし）
    ```bash
    clpplus -nw "${DB_USER}/${DB_PASS}@${DB_HOST}:${DB_PORT}/${DB_NAME}" <<EOF
    SET SERVEROUTPUT ON;
    @hello_world.sql
    DISCONNECT;
    EXIT;
    EOF
    ```

> [!NOTE]
>シェルスクリプトからPL/SQLを実行して任意の完了コードを受け取りたい場合（業務的な意味を持つ完了コードを設計する場合）は、  
>バインド変数とPRINT(バインド変数を表示させるコマンド)を使って任意の完了コードを返すように記述することが出来る。  
>ヒアドキュメントにPL/SQLをべた書きする場合の例を以下に示す（SQLファイルとして実行する場合も同様）。  
>```bash
># 「-s」(サイレントモード)を指定すると基本的なメッセージが抑止される＆DISCONNECTやEXITを記述しなくても終了できる。
># そのうえで抑止しきれない標準出力をSETコマンドでOFFにし、PRINT結果のみを標準出力経由で変数RESULT_CODEに格納できるようにする。
>RESULT_CODE=$(clpplus -s -nw "${DB_USER}/${DB_PASS}@${DB_HOST}:${DB_PORT}/${DB_NAME}" <<EOF
>-- 出力フォーマットの調整（ヘッダーや余計な空白を消す）
>SET PAGESIZE 0;
>SET FEEDBACK OFF;
>SET VERIFY OFF;
>SET TERMOUT ON;
>
>-- バインド変数の宣言
>VARIABLE exit_code NUMBER;
>
>-- PL/SQLブロックの実行
>BEGIN
>  -- ここで何らかの処理を行う
>  :exit_code := 100 + 200;
>END;
>/
>
>-- バインド変数の値を標準出力に書き出す
>PRINT exit_code;
>EXIT;
>EOF
>)
>
>RESULT_CODE=$(echo $RESULT_CODE | xargs)
>echo "PL/SQLから受け取った値は: ${RESULT_CODE}"
>```

## 4-3.よく使用する設定

SETコマンドの全量についてはIBM公式サイトを参照すること。  
https://www.ibm.com/docs/ja/db2/12.1.x?topic=commands-set

* SET ECHO OFF  
コマンドとその出力の表示を使用不可にする。
* SET PAGESIZE 0  
検索結果のヘッダー行（カラム名）を非表示にする。
* SET FEEDBACK OFF  
「DB250000I: The command completed successfully.」という様なメッセージを抑止する。
* SET SERVEROUTPUT ON  
「DBMS_OUTPUTパッケージ」を利用した標準出力をONにする。原則としてデバッグ目的で利用すべきであり常用は非推奨。
* SET TERMOUT ON  
標準出力を有効にする。サイレントモードかつPRINTコマンドで標準出力したいというケースで主に使用（それ以外では原則指定する必要なし）。