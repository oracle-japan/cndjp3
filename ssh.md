SSHによる一時借し出し環境へのアクセス手順
=========================================
注意）この環境を利用される方は、はじめに講師から以下の情報について案内を受けて下さい。

- クラウド上のLinuxマシンのIPアドレス
- SSHアクセス用の鍵ファイル


0 . 事前準備
------------
ハンズオンチュートリアルのドキュメント一式を、任意のgitクライアントでcloneしておきます。
以下は、コマンドラインのgitクライアントを使う場合の例です。

    > git clone https://github.com/oracle-japan/cndjp3.git

次に、cloneしてできたディレクトリをカレントディレクトリにしておきます。

    > cd cndjp3


1 . 環境へのアクセス
--------------------

### OpenSSHを使う場合
以下のコマンドを実行します。

    > ssh -i .\keys\rsa-key-20171213.ssh opc@129.146.85.39

### PuTTYを使う場合（Windowsのみ）
PuTTYを起動して、以下の設定で新規セッションを作成して下さい（指定のない箇所は全てデフォルト値）。

・Session > Host Name: opc@[環境のIPアドレス]

![](images/PuTTY_Configuratio1.png "Host Name")

・Connection > SSH > Private key file for authentication: keys/rsa-key-20171213.ppk

![](images/PuTTY_Configuratio2.png "Private key filr for authentication")

