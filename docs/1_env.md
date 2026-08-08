# 1.環境構築

前提：Db2 v12.1のCommunity Edition

## 1-1. Db2のインストール

1. IBMのサイトからインストーラをダウンロードする（`v12.1.2_linuxx64_server_dec.tar.gz`）
2. tar.gzを展開する
    ```sh
    tar -zxvf v12.1.2_linuxx64_server_dec.tar.gz
    ```
3. 展開したディレクトリ（`server_dec`）に移動する
    ```sh
    cd server_dec
    ```
4. インストール前チェックを実行する
    ```sh
    ./db2prereqcheck
    ```
    > [!NOTE]
    >依存関係に不足があれば以下の様に表示される。  
    >インストールする機能によっては不要でよい依存関係が含まれている場合があるので、ネットで検索して取捨選択する。  
    >DBT3622E  The db2prereqcheck utility failed to verify the prerequisites for PCMK. Ensure your machine meets all the PCMK installation prerequisites.  For detailed information, please check /tmp/db2prereqPCMK.log.*   
    >DBT3507E  The db2prereqcheck utility failed to find the following package or file: "kernel-devel".   
    >DBT3507E  The db2prereqcheck utility failed to find the following package or file: "gcc-c++".   
    >DBT3507E  The db2prereqcheck utility failed to find the following package or file: "cpp".   
    >DBT3507E  The db2prereqcheck utility failed to find the following package or file: "gcc".   
    >DBT3563E  The db2prereqcheck utility determined that SELinux is enabled, which is not supported with IBM Storage Scale.   
    >DBT3618E  The db2prereqcheck utility detected that ksh is not linked to ksh, ksh93 or mksh or lksh. This is required for Db2 High Availability Feature.   
5. インストールを実行する
    ```sh
    sudo ./db2_install
    ```
    > [!NOTE]
    >対話応答を求められるものはデフォルト値での応答でよい。  
    >ただし「pureScale」機能については「no」で応答すること。

## 1-2. ユーザーの作成

1. Db2管理用のユーザーを作成する（必須）
    ```sh
    sudo groupadd -g 989 db2iadm1
    sudo groupadd -g 988 db2fadm1
    sudo groupadd -g 987 dasadm1
   
    sudo useradd -u 1004 -g db2iadm1 -m -d /home/db2inst1 db2inst1
    sudo useradd -u 1003 -g db2fadm1 -m -d /home/db2fenc1 db2fenc1
    sudo useradd -u 1002 -g dasadm1 -m -d /home/dasusr1 dasusr1
   
    sudo passwd db2inst1
    sudo passwd db2fenc1
    sudo passwd dasusr1
    ```
    > [!NOTE]
    >グループIDおよびユーザーIDは既存のものと重複しなければ任意のものでよい。

    > [!NOTE]
    >この3つのユーザーは常用すべきものではない。

2. Db2利用用のユーザーを作成する（必要に応じて）
    ```sh
    sudo groupadd -g nnn 
    sudo useradd -u mmm -g XXXXX -m -d /home/YYYY YYYY
    ```
    > [!NOTE]
    >Db2の権限設定はグループ単位・ユーザー単位のどちらでも指定できる。  
    >その関係でグループ名・ユーザー名の横断でユニークな名称になっている必要がある。  
    >グループ指定無しでユーザー作成した場合は「ユーザー名＝グループ名」として作成されるため、  
    >権限設定時にグループとユーザーのどちらが指定されているのかが特定できず、エラーになってしまう。  
    >この問題を回避するためにグループを明示的に作成・指定してユーザーを作成する必要がある。

## 1-3. インスタンスの作成

1. インスタンス(db2inst1)を作成
    ```sh
    sudo /opt/ibm/db2/V12.1/instance/db2icrt -u db2fenc1 db2inst1
    ```
2. 作成したインスタンスの通信ポートを確認
    ```sh
    cat /etc/services
    ```
    例：以下のように表示された場合はdb2c_db2inst1の25010が通信ポート。
    >DB2_db2inst1    20016/tcp  
    >DB2_db2inst1_1  20017/tcp  
    >DB2_db2inst1_2  20018/tcp  
    >DB2_db2inst1_3  20019/tcp  
    >DB2_db2inst1_4  20020/tcp  
    >DB2_db2inst1_END        20021/tcp  
    >db2c_db2inst1   25010/tcp

    > [!NOTE]
    >ポート番号はインスタンス作成時に指定したり後から変更することも可能。詳細はネットで検索すること。

## 1-4. データベースの作成

> [!NOTE]
>インスタンスを起動(db2start)してから実行する。

1. Oracle互換モードを使用する場合（使用しない場合はSKIP）
    ```sh
    su - db2inst1
    db2set DB2_COMPATIBILITY_VECTOR=ORA
    exit
    ```
    > [!NOTE]
    >DB2_COMPATIBILITY_VECTORの設定変更を反映するためにインスタンスの再起動が必要（db2stop & db2start）。

2. データベースを作成する
    ```sh
    su - db2inst1
    db2 "create database XXXX using codeset utf-8 territory jp collate using identity"
    exit
    ```

## 1-5. スキーマの作成

> [!NOTE]
>インスタンスを起動(db2start)してから実行する。

1. スキーマを作成する
    ```sh
    su - db2inst1
    db2 connect to XXXX
    db2 create schema YYYY
    exit
    ```

## 1-6. データベースの自動起動設定

1. 手動起動
    ```sh
    su - db2inst1
    db2start
    exit
    ```

2. 自動起動設定
    2-1. 以下のコマンドを実行する
    ```sh
    sudo /opt/ibm/db2/V12.1/bin/db2fmcu -u -p /opt/ibm/db2/V12.1/bin/db2fmcd
    ```

    2-2. インスタンスオーナー（ユーザーdb2inst1）で以下のコマンドを実行する
    ```sh
    su - db2inst1
    db2iauto -on db2inst1
    db2set DB2AUTOSTART
    ```
    > [!NOTE]
    >手順2-2の結果を2-3で確認する。`db2set DB2AUTOSTART`を実行した時点では特に応答メッセージは表示されない。

    2-3. 設定結果の確認①（ユーザーdb2inst1で続けて実行）：
    ```sh
    db2greg -getinstrec instancename=db2inst1 | grep -i StartAtBoot
    ```
    > [!NOTE]
    > 以下のようになっていればOK。
    > ```bash
    >   StartAtBoot  = 1
    > ```

    2-4. 設定結果の確認②（ユーザーdb2inst1で続けて実行）：
    ```sh
    db2 get dbm cfg | grep SVCENAME
    ```
    > [!NOTE]
    > 以下のようになっていればOK。
    > ```bash
    > TCP/IP Service name                          (SVCENAME) = db2c_db2inst1  
    > SSL service name                         (SSL_SVCENAME) =
    > ```

## 1-7. JDBC接続設定

1. 基本形
    ```
    jdbc:db2://${hostName}:${portNo}/${dbName}:currentSchema=${schemaName};
    ```
2. 応用形（CURRENT PATH 特殊レジスターの指定）
    ```
    jdbc:db2://${hostName}:${portNo}/${dbName}:currentSchema=${schemaName};currentFunctionPath=${schemaName},SYSIBM,SYSFUN,SYSPROC,SYSIBMADM;
    ```
    > [!NOTE]
    > CURRENT PATH 特殊レジスターについては下記ページを参照。  
    > [9-7-1.「CURRENT_SCHEMA」を指定してもエラーになる理由](./9_tips.md#9-7-1current_schemaを指定してもエラーになる理由)  
