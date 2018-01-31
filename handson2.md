CNDJP #3 ハンズオンチュートリアル Part 2
========================================

これは、cndjp第3回勉強会のハンズオン Part 2のチュートリアルです。

このチュートリアルでは、Part 1でKubernetes上にデプロイしたチャットアプリケーションをに対して、カオステストを実施します。


前提条件
--------
このチュートリアルは、[cndjp第3回勉強会のハンズオンチュートリアル Part 1](https://github.com/oracle-japan/cndjp3/blob/master/handson1.md)を実施済みであることを前提とします。

以降の手順は、ほとんどの操作をコマンドラインツールから実行します。Mac/Linuxの場合はターミナル、Windowsの場合はWindows PowerShell利用するものとして、手順を記述します。


0 . 準備作業
------------
このリポジトリをcloneしてできたディレクトリをカレントディレクトリにします。

    > cd ./cndjp3/

Kafkaクラスターを含む、チャットアプリケーションのPod群が稼働してることを確認しておきます。

    > kubectl get pods
    NAME                                      READY     STATUS    RESTARTS   AGE
    kafka-0                                   1/1       Running   0          19m
    kafka-1                                   1/1       Running   0          18m
    kafka-2                                   1/1       Running   0          18m
    kafka-3                                   1/1       Running   0          18m
    kafka-client                              1/1       Running   0          15m
    kchat-server-kconsumer-4076213875-fl7pr   1/1       Running   0          10m
    kchat-server-kproducer-2136001974-vzltw   1/1       Running   0          10m
    pzoo-0                                    1/1       Running   0          24m
    pzoo-1                                    1/1       Running   0          23m
    pzoo-2                                    1/1       Running   0          23m
    zoo-0                                     1/1       Running   0          24m
    zoo-1                                     1/1       Running   0          23m

チャットクライアントを起動しておきます。

    > java -jar kchat.client-jar-with-dependencies.jar 172.17.8.102 "User A"

新たにターミナル/PowerShellを立ち上げ、このリポジトリをcloneしてできたディレクトリ配下にある`chaos-testing`を、カレントディレクトリにします。

    > cd ./cndjp3/chaos-testing

以降の操作は、特に断りのない場合はこのコンソールで作業をします。


1 . ロードジェネレーターを準備する
----------------------------------
運用中のリクエストの流入を再現するためのツールとして、ロードジェネレーターを利用します。ここではApache Benchを利用します。

Apache BenchはApach HTTPサーバーに付属しますので、それをインストールします。Macの場合は標準でインストール済みのため、この作業は不要です。

・Linux (yumベースの場合の例）

    > sudo yum install httpd 

・Windows

    > [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
    > Invoke-WebRequest -Uri https://www.apachelounge.com/download/VC15/binaries/httpd-2.4.29-Win64-VC15.zip -OutFile httpd-2.4.29-Win64-VC15.zip
    > Expand-Archive -Path .\httpd-2.4.29-Win64-VC15.zip

続いてリクエストの送信を行います。

・Mac/Linux

    > ab -n 10 -c 1 -p ab-test-message.txt http://172.17.8.102:30192/api/topics/chatroom

・Windows
    > .\httpd-2.4.29-Win64-VC15\Apache24\bin\ab.exe -n 10 -c 1 -p .\ab-test-message.txt http://172.17.8.102:30192/api/topics/chatroom

チャットクアイラントで、メッセージを10件受信できていることを確認します。

これでロードジェネレーターの準備は完了です。


2 . 障害再現ツールを実行する
----------------------------
障害を再現するためのツールとして、[chaoskube](https://github.com/linki/chaoskube)を利用します。これは、Kubernetesクラスター上のPodをランダムにダウンさせるものです。

まず、新たにターミナル/PowerShellを立ち上げ、`chaos-testing`を、カレントディレクトリにします。

    > cd ./cndjp3/chaos-testing

### 2-1. Helmの準備
chaoskubeはHelm Chartとして提供されており、KubernetesクラスターにデプロイするにはHelmを利用します。<br>
Helmは複数のmanifestをパッケージとして管理するためのツールで、複数のコンテナをセットにしてデプロイするようなユースケースにおいて利用が広がっているものです。

Helmをインストールするには、[GitHubのReleases](https://github.com/kubernetes/helm/releases)から、自分の環境に合ったものをダウンロードし、PATHを通しておきます。

ただし、Mac/Linuxの場合は、他の手段も提供されています。

・Mac
    
    > brew install kubernetes-helm

・Mac/Linux

    > $ curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
    $ chmod 700 get_helm.sh
    $ ./get_helm.sh

インストールが完了したら、`helm init`で初期化しておきます。この操作で、クラスター内に常駐するHelm Tillerが配備されます。<br>
対象のクラスターはkubectlの設定情報から自動的に識別してくれます。

    >helm init
    $HELM_HOME has been configured at C:\Users\hhayakaw\.helm.
    
    Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.
    Happy Helming!

### 2-2. chaoskubeを実行する
chaoskubeをクラスターにデプロイし、定期的にPodのダウンを発生させます。以下のコマンドを実行してください。

    > helm install stable/chaoskube --version 0.6.1 --set interval=1m,dryRun=false,labels=app=kafka

このコマンドで与えているパラメータは、以下のような意味です。ここに記述する以上のオプションの詳細は、[chaoskube](https://github.com/linki/chaoskube)のドキュメントを参照してください。

- interval=1m
    * 1分おきにPodをダウンさせる。デフォルトは10分
- dryRun=false                    
    * dry runモード（Podを落とさない）を無効にする。デフォルトはTRUE
- labels=app=kafka                 
    * ダウンさせる対象のPodを絞るlabelの指定。この場合kafkaのbroker群が対象となる

コマンドの実行結果として返ってくるメッセージに、chaoskubeの動作状況をtailするためのコマンドが記載されていますので、そのコマンドを実行します。

    > POD=$(kubectl get pods -l app=mewing-gibbon-chaoskube --namespace kafka --output name)
    > kubectl logs -f $POD --namespace=kafka

Windowsの場合は、上記の例の場合は以下のように実行してください。

    > kubectl get pods -l app=mewing-gibbon-chaoskube --namespace kafka --output name
    > kubectl logs -f [上のコマンドの実行結果] --namespace=kafka

以下のように、1分おきにbrokerのPodをダウンさせている様子が確認できます。

    time="2018-01-31T04:03:37Z" level=info msg="Targeting cluster at https://10.100.0.1:443"
    time="2018-01-31T04:03:39Z" level=info msg="Filtering pods by labels: app=kafka"
    time="2018-01-31T04:03:50Z" level=info msg="Killing pod kafka/kafka-1"
    time="2018-01-31T04:04:53Z" level=info msg="Killing pod kafka/kafka-0"
    time="2018-01-31T04:05:57Z" level=info msg="Killing pod kafka/kafka-2"


3 . アプリケーションの挙動を確認する
------------------------------------
chaoskubeを実行させている状況下で、アプリケーションの挙動がどうなるかを観察してみて見ます。

### Partitionの再配置の様子を観察する
Podのダウン発生、及びその後の自動復旧（Podの復旧自体はkubernetesの機能です）の前後で、Partitionの再配置（リバランス）が行われる様子を確認します。

これを行うには、kafka-clientからTopicの情報を出力するコマンドを繰り返し実行します。

    > kubectl exec -it kafka-client bash
    root@kafka-client:/opt/kafka# ./bin/kafka-topics.sh --describe --zookeeper zookeeper.kafka:2181 --topic chatroom

### 障害発生／復旧中のメッセージ配信の挙動を観察する
chaoskubeを動かしたまま、Apache Benchでリクエストを送信し続け、チャットクライアントでのメッセージ受信の遅延がどのように発生するか、確認します。

環境によって異なりますが、Apache Benchで1000リクエスト程度を直列に送るようにすると、1分強程リクエストが送信され続ける状況を作ることができます。

・Mac/Linux

    > ab -n 1000 -c 1 -p ab-test-message.txt http://172.17.8.102:30192/api/topics/chatroom

・Windows

    > .\httpd-2.4.29-Win64-VC15\Apache24\bin\ab.exe -n 1000 -c 1 -p .\ab-test-message.txt http://172.17.8.102:30192/api/topics/chatroom


（あとはお好きなようにいじってみてください…。）


以上で、ハンズオンのPart2は終了です。
