# 5.データ入出力ユーティリティ

## 5-1. 比較

|ユーティリティ|方式|速度|主な用途|テーブルへの影響|
|---|---|---|---|---|
|EXPORT|SELECT|高速|ファイルへのデータ抽出|なし（読み取りのみ）|
|IMPORT|INSERT/UPDATE|低速|大量更新を安全に実施|大きい（行レベルロック）|
|LOAD|直接ページ書込|最高速|大規模データの初期投入|非常に大きい（排他ロック）|
|INGEST|ストリーム挿入|高速|24時間稼働環境でのデータ更新|小さい（行レベルロック）|

> [!NOTE]
> LOADは最高速でINSERTやINSERT_UPDATE、REPLACEを実施できるがトランザクションログを書き込まないため、ロールフォワードのために表スペースのバックアップが必要となる（バックアップを取得するまで"バックアップ・ペンディング"状態となり、表を利用できなくなる）。  
> "バックアップ・ペンディング"状態になるのを回避するためにはNONRECOVERABLEを指定すれば良いが、データ損失が発生した場合は復旧不能になるので注意が必要である。  

> [!NOTE]
>上記の"トランザクションログを書き込まない"性質に関連して、トランザクションログを利用してレプリケーション・HA構成となっているデータベースの表に対してLOADを実行した場合、データが同期されないという問題が生じる。  
>原則として本番運用中のデータベースに対してLOADを実行するべきではない。

## 5-2. EXPORT

### 基本形（カンマ区切り・ダブルクォート囲み・UTF-8）
```sql
EXPORT TO [出力ファイルパス] OF DEL SELECT * FROM [テーブル名] WHERE [抽出条件]
```
> [!NOTE]
>"OF DEL" ＝ カンマ区切り

### 応用形（TSV形式・ダブルクォート囲み・Windows-31J）
```sql
EXPORT TO [出力ファイルパス] OF DEL MODIFIED BY coldel0x09 codepage=943 SELECT * FROM [テーブル名] WHERE [抽出条件]
```
> [!NOTE]
>"OF DEL MODIFIED BY coldel0x09" ＝ フィールド区切り文字をタブにする  

> [!NOTE]
>"codepage=943" ＝ Windows-31J(MS932)  
>指定を省略した場合は `codepage=1208`(UTF-8) として扱われる。

## 5-3. IMPORT

### 基本形

```sql
IMPORT FROM [入力ファイルパス] OF DEL COMMITCOUNT=[コミット間隔] INSERT INTO [テーブル名]
```
> [!NOTE]
> IMPORTの動作モードは下表のとおり（LOADでも同様のモード指定が可能）。
> |モード|動作|
> |---|---|
> |INSERT INTO|主キー重複が無ければINSERTする|
> |INSERT_UPDATE|主キー重複が無ければINSERTし、主キー重複が有ればUPDATEする|
> |REPLACE|全レコードをDELETEしてからINSERTする|

### 応用形
```sql
IMPORT FROM [入力ファイルパス] OF DEL MODIFIED BY codepage=943 chardel0x1f coldel0x09 delprioritychar COMMITCOUNT=[コミット間隔] INSERT INTO [テーブル名]
```
> [!NOTE]
>"codepage=943" ＝ Windows-31J(MS932)

> [!NOTE]
>"chardel0x1f" ＝ フィールド囲み文字を「ユニット区切り」として解釈する  
>
>この例は、ダブルクォート以外が囲み文字として指定されているCSVデータを取込する際の例である。  
>「ユニット区切り」とはASCIIコードに含まれる制御コードの一つ。  
>一般的には、データにダブルクォートを含む場合はエスケープすることで囲み文字のダブルクォートと区別できるようにするが、  
>EXPORTする側の都合等に起因してエスケープが不可能な場合の代替策として、  
>PCの通常利用では入力されることが無いASCII制御コードを囲み文字として指定してEXPORTする方法がある。  
  
> [!NOTE]
>coldel0x09 ＝ フィールド区切り文字を「タブ」として解釈する  

> [!NOTE]
>"delprioritychar" ＝ 改行コードよりも囲み文字の組み合わせを優先する(改行コードを含むデータを扱う場合に有力なオプション)

## 5-4. LOAD

### 前提

* codepage、chardel、coldel、delprioritycharはIMPORTと同様に指定が可能。
* COMMITCOUNTは指定出来ない（通常のトランザクションログをバイパスし、ページレベルでの一括処理のため）。
* "バックアップ・ペンディング"状態が発生する（NONRECOVERABLEを指定しない場合）。
* NONRECOVERABLEを指定してデータ損失が発生した場合に復旧不可能になる（テーブルの再作成およびCSVデータ等を用いた復旧しか出来ない）。
* 本番環境に対してLOADを利用するべきではない（IMPORTもしくはINGESTを利用するべき）。
* システムの本番稼働前に大量データの初期登録を行う様な場合でのみ利用すること。

### リスクを承知の上で利用する方法
```sql
LOAD FROM [入力ファイルパス] OF DEL INSERT INTO [テーブル名] NONRECOVERABLE
```

### LOAD実行状況を確認する
```sql
list utilities show detail
```

### LOADでエラーが発生した場合のトラブルシュート

#### ロードペンディング状態を確認＆前進

1. [9-5-1.テーブルの状態確認](9_tips.md#9-5-1テーブルの状態確認) を実施する。
2. LOAD_STATUS が 「IN_PROGRESS」の場合はLOAD処理が進行中、「PENDING」になっていたらペンディングになってしまっている。
3. NO_LOAD_RESTART が「Y」になってしまっていたらLOAD再始動不可になってしまっている。  
「N」の場合はLOAD RESTARTが可能。  
※「NONRECOVERABLE」を指定してLOADを起動した場合はLOAD RESTARTは不可能なので、TERMINATEさせて原因解決後にイチからLOADしなおす必要がある。
4. エラーを発生させてしまったLOAD文の「INSERT」「INSERT_UPDATE」「REPLACE」の部分を「TERMINATE」に書き換えて再実行する。  
これによってペンディング状態になってしまっているLOAD処理を終了させ、テーブルに対する操作を可能にする。
5. LOADがエラーになった原因を解消した後にLOADをリトライする。

#### バックアップペンディング状態を確認＆前進

1. バックアップペンディング状態になっているかどうかを確認する

    ```bash
    # 構成パラメータから確認する
    db2 get db cfg for [データベース名] | grep backup_pending
    ```
    ```bash
    # 表スペースの状態から確認する（Backup pendingとなっている表スペースがあるかを確認する）
    db2 list tablespaces show detail
    ```

2. バックアップ実行
    ```bash
    # NONRECOVERABLEを指定してLOADすべきケースで指定忘れしてしまったような場合は（＝当該表を洗い替える場合等）、
    # nullデバイスにバックアップデータを捨てることで、形式上のバックアップを実行してペンディング状態を解消させる。
    # ！！本来はきちんとしたバックアップを取得すべき！！
    db2 backup database [データベース名] to /dev/null
    ```

## 5-5. INGEST

IBM公式サイトを参照  
https://www.ibm.com/docs/ja/db2/12.1.x?topic=commands-ingest
