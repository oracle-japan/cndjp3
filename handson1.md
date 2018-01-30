CNDJP #3 ハンズオンチュートリアル
=================================

これは、cndjp第3回勉強会のハンズオン Par1のチュートリアルです。

このチュートリアルでは、[Apache Kafka](https://kafka.apache.org/)を利用したチャットアプリケーションを、Kubernetes上にデプロイします。

<!-- 画像 -->

Kafkaは、2011年にLinkedInから公開されたオープンソースの分散メッセージングシステムで、大量のストリーミングデータ（ログ、イベントなど）を収集／配信する基盤として広く利用されています。


前提条件
--------
（cndjp第3回勉強会の現地で、運営側がお借しする環境を利用される方は「前提条件（一時借し出し環境向け）」を参照下さい。

このチュートリアルは、[cndjp第1回勉強会のハンズオンチュートリアル](https://github.com/oracle-japan/cndjp1/blob/master/handson/handson1.md)の「2. kubectlのセットアップ」以降までを実施済みであることを前提とします。

以降の手順は、ほとんどの操作をコマンドラインツールから実行します。Mac/Linuxの場合はターミナル、Windowsの場合はWindows PowerShell利用するものとして、手順を記述します。


前提条件（一時借し出し環境向け）
--------------------------------
一時借し出し環境では、クラウド上のLinuxサーバーにVirtualboxの仮想マシンを稼働させ、それらの仮想マシンをノードとしてKubernetesクラスターを稼働させています。

    --------------------------
    |      k8s cluster       |  <-- (管理操作)
    --------------------------          |
    |   vm   |   vm  |   vm  |          |
    --------------------------     -----------
    | Hypervisor(VirtualBox) |     | kubectl |
    ------------------------------------------
    |      OS（クラウド上のLinuxマシン）     |
    ------------------------------------------

上の図で、OSより上位のレイヤーについては、cndjp第1回勉強会のハンズオンチュートリアルで作成するものと同等です。

環境へのアクセスにはSSHを利用します。お好きなSSHクライアントをお使いください。
SSHを利用した環境へのアクセス手順については、「[SSHによる一時借し出し環境へのアクセス手順](./ssh.md)」を参照下さい。


0 . 準備作業
------------

### 0-1. Kubernetesクラスターへのアクセスの確認
Kubernetesクラスターを停止している方は、ここで起動をしておいて下さい。

まず、Kubernetesのインストールスクリプトが配置されたディレクトリをカレントディレクトリにします。

    > cd [インストールスクリプトが配置されたディレクトリ]

一時借し出し環境の場合、以下のようになります。

    > cd ~/kubernetes-vagrant-coreos-cluster/


クラスターが起動しているかどうか確認するため、`vagrant status`を実行します。<br>
以下のような結果が返る場合、クラスターは停止しています。

    > vagrant status
    Current machine states:

    master                    poweroff (virtualbox)
    node-01                   poweroff (virtualbox)
    node-02                   poweroff (virtualbox)

    This environment represents multiple VMs. The VMs are all listed
    above with their current state. For more information about a specific
    VM, run `vagrant status NAME`.

クラスターを起動するには、以下のコマンドを実行します。

    > vagrant up
    Bringing machine 'master' up with 'virtualbox' provider...
    Bringing machine 'node-01' up with 'virtualbox' provider...
    Bringing machine 'node-02' up with 'virtualbox' provider...
    ==> master: Running triggers before up...
    ==> master: 2017-11-15 01:25:08 +0900: setting up Kubernetes master...
    ...（中略）
        node-02: Running: inline script
    ==> node-02: Running triggers after up...
    ==> node-02: Waiting for Kubernetes minion [node-02] to become ready...
    ==> node-02: 2017-11-15 01:45:39 +0900: successfully deployed node-02

クラスターを起動したら、以下のコマンドを実行してクラスターへの疎通を確認します。

・クラスター情報の取得

    > kubectl cluster-info
    Kubernetes master is running at https://172.17.8.101
    KubeDNS is running at https://172.17.8.101/api/v1/namespaces/kube-system/services/kube-dns/proxy

    To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

・ノードの一覧の取得

    > kubectl get nodes
    NAME           STATUS                     AGE       VERSION
    172.17.8.101   Ready,SchedulingDisabled   12h       v1.7.11
    172.17.8.102   Ready                      12h       v1.7.11
    172.17.8.103   Ready                      12h       v1.7.11


### 0-2. チュートリアルのファイル一式を取得する
アプリケーションの一連のmanifestファイルが、このチュートリアルをホストするリポジトリに保存されています。まずはこのリポジトリを適当なディレクトリにcloneして下さい。

以下は、コマンドラインツールのgitを使う例です。

    > git clone https://github.com/oracle-japan/cndjp3.git


1 . KubernetesにKafkaクラスターをデプロイする
---------------------------------------------
一連のmanifestファイルはdeploymnetフォルダ配下に配置してあります。まずはこのフォルダをカレント・ディレクトリにしておきます。

    > cd cndjp3/deployment

### 1-1. 永続化領域を構成する
Kafkaはメッセージを保持するBrokerと、それを管理するzookeeperのそれぞれにおいて、データを永続化する領域を必要とします。はじめに、この領域のためのPersistentVolumeを構成します。

manifestファイルは`./kafka/pv`に作成してあります。例えばBroker用のPersistentVolumeのmanifestを参照してみます。

    > ls ./kafka/pv
    > cat ./kafka/pv/pv-hostpath-broker.yml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: kafka-broker-0
spec:
  capacity:
    storage: 3Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/kafka/broker/0
    # type: DirectoryOrCreate
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: kafka-broker-1
…
```

今回は、4つのBrokerをによるクラスターを構成しますので、4つのPVを記述しています。動的にBrokerを追加したい場合には、この例のような固定のPVではなく、[ボリュームの動的プロビジョニング](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)を利用します。

それでは、このSecretをクラスター上に作成します。

    > kubectl create -f ./kafka/pv/
    persistentvolume "kafka-broker-0" configured
    persistentvolume "kafka-broker-1" configured
    persistentvolume "kafka-broker-2" configured
    persistentvolume "kafka-broker-3" configured
    persistentvolume "kafka-zookeeper-0" configured
    persistentvolume "kafka-zookeeper-1" configured
    persistentvolume "kafka-zookeeper-2" configured

作成されたをPersistentVolumeを確認してみます。

・一覧取得

    > kubectl get pv

・詳細情報取得

    > kubectl describe pv/kafka-broker-0


### 1-2. zookeeperのクラスターをデプロイする
つづいてzookeeperをデプロイします。<br>
zookeeperに関連するmanifestファイルの一式が`./kafka/zookeeper`に作成してあります。zookeeper本体と、それにアクセスする入り口となるServiceオブジェクトなどのmanifestがあります。ここではzookeeper本体のmanifestの中身を確認してみます。

    > ls ./kafka/zookeeper
    > cat ./kafka/zookeeper/50pzoo.yml

```yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: pzoo
  namespace: kafka
spec:
  serviceName: "pzoo"
  replicas: 3
  template:
    metadata:
      labels:
        app: zookeeper
        storage: persistent
      annotations:
    spec:
      terminationGracePeriodSeconds: 10
      initContainers:
          …
      containers:
      - name: zookeeper
        image: solsson/kafka:1.0.0@sha256:17fdf1637426f45c93c65826670542e36b9f3394ede1cb61885c6a4befa8f72d
        env:
        - name: KAFKA_LOG4J_OPTS
          value: -Dlog4j.configuration=file:/etc/kafka/log4j.properties
        command:
        - ./bin/zookeeper-server-start.sh
        - /etc/kafka/zookeeper.properties
        ports:
        - containerPort: 2181
          name: client
        - containerPort: 2888
          name: peer
        - containerPort: 3888
          name: leader-election
        resources:
          requests:
            cpu: 10m
            memory: 100Mi
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - '[ "imok" = "$(echo ruok | nc -w 1 -q 1 127.0.0.1 2181)" ]'
        volumeMounts:
        - name: config
          mountPath: /etc/kafka
        - name: data
          mountPath: /var/lib/zookeeper/data
      volumes:
      - name: config
        configMap:
          name: zookeeper-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          # storage: 10Gi
          storage: 1Gi

```

ポイントはKindでStatefulSetを指定していることです。

StatefulSetは、Deploymentオブジェクト同様、`spec.replicas`で指定された数のPodを起動するものです。Deploymentとの違いは、それぞれのPodに連番の名前が与えられ、永続領域とネットワーク上の名前がPod毎に管理されます。Podを停止／再起動しても、以前と同じ永続領域とネットワーク名が利用されます。

Kafka以外にも、MySQLのクラスターなど、ステートフルなPodを利用するケースではStatefulSetを利用します。

それでは、zookeeperのコンテナをデプロイします。

    > kubectl create -f ./kafka/zookeeper/
    namespace "kafka" created
    configmap "zookeeper-config" created
    service "pzoo" created
    service "zoo" created
    service "zookeeper" created
    statefulset "pzoo" created
    statefulset "zoo" created

ここでは"kafka"という名前のNamespaceを作成しています。Namespaceはクラスター内のオブジェクトを分類して扱うための仕組みです。Namespaceを指定してkubectlコマンドを利用することで、そのコマンドの効力をNamespace内のオブジェクトに限定する事ができます。

以降、このNamespaceにオブジェクトを作成していくので、kubectlのデフォルトのNamespaceを変更しておきます。

・Mac/Linux

    > kubectl config set-context $(kubectl config current-context) --namespace=kafka

・Windows

    > kubectl config current-context
    default-cluster
    > kubectl config set-context default-cluster --namespace=kafka
    Context "default-cluster" modified.

zookeeperのPodのデプロイが完了するまで1分程度かかるので、ここで少し待ちます。`kubectl get pod`の結果が以下のようになれば、完了です。

    > kubectl get pod
    NAME      READY     STATUS    RESTARTS   AGE
    pzoo-0    1/1       Running   0          1m
    pzoo-1    1/1       Running   0          36s
    pzoo-2    1/1       Running   0          21s
    zoo-0     1/1       Running   0          1m
    zoo-1     1/1       Running   0          1m


### 1-3. 永続化領域を構成する
つづいてbrokerをデプロイします。<br>
brokerに関連するmanifestファイルの一式は`./kafka/`配下のymlファイル群です。broker本体と、それにアクセスする入り口となるServiceオブジェクトなどのmanifestがあります。

broker本体のmanifestの中身を確認してみます。

    > ls ./kafka/
    > cat ./kafka/50kafka.yml

```yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: kafka
  namespace: kafka
spec:
  serviceName: "broker"
  replicas: 4
  template:
    metadata:
      labels:
        app: kafka
      annotations:
    spec:
      terminationGracePeriodSeconds: 30
      initContainers:
        …
      containers:
      - name: broker
        image: solsson/kafka:1.0.0@sha256:17fdf1637426f45c93c65826670542e36b9f3394ede1cb61885c6a4befa8f72d
        env:
        - name: KAFKA_LOG4J_OPTS
          value: -Dlog4j.configuration=file:/etc/kafka/log4j.properties
        ports:
        - containerPort: 9092
        command:
        - ./bin/kafka-server-start.sh
        - /etc/kafka/server.properties
        - --override
        -   zookeeper.connect=zookeeper:2181
        - --override
        -   log.retention.hours=-1
        - --override
        -   log.dirs=/var/lib/kafka/data/topics
        - --override
        -   auto.create.topics.enable=false
        resources:
          requests:
            cpu: 100m
            memory: 512Mi
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - 'echo "" | nc -w 1 127.0.0.1 9092'
        volumeMounts:
        - name: config
          mountPath: /etc/kafka
        - name: data
          mountPath: /var/lib/kafka/data
      volumes:
      - name: config
        configMap:
          name: broker-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          # storage: 200Gi
          storage: 2Gi
```

StatefulSetを使っているのは、zookeeperの場合と同様です。今回の構成の場合、Kafkaは受け取ったメッセージを永続化するため、各brokerに対応する永続化領域が必要になります。

また、zookeeperの場合もそうでしたが、各brokerに与えるプロパティは、broker-configという名前のConfigMapオブジェクトから受け取っています。brokerがzookeeperにアクセスするためのアドレスや、クラスター間のデータリバランス（Part2にて後述）の設定など、brokerに関連する設定値がこのオブジェクトから与えられます。

次に、Serviceオブジェクトのmanifestを見てみます。

    > cat ./kafka/20dns.yml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: broker
  namespace: kafka
spec:
  ports:
  - port: 9092
  clusterIP: None
  selector:
    app: kafka
```

`spec.ports.clusterIP`に"None"を指定しており、こうするとHeadress Serviceというオブジェクトが構成されます。

Headress Serviceは、そのServiceが対象とするPod（この場合はbroker群）に対し、FQDNで直接アクセスすることを許可するものです。ここでのFQDNは、Kubernetesクラスター内のPod群に対して割当たるDNSで、同じくクラスター内で稼働するkube-dnsによって名前解決されます。<br>
通常のServiceではkube-proxy経由でPodにルーティングされるため、直接固定のPodにアクセスできる保証がありません。しかしHeadress Serviceでは、アドレスを指定すれば目的のPodに直接アクセスできます。

Headress Serviceを使うかどうかは、利用するソフトウェア（この場合はKafkaのbroker）の要件に合わせて選択します。

それでは、brokerのコンテナをデプロイします（一部エラーが出ますが、このあとの手順に影響はありません）

    > kubectl create -f ./kafka/
    configmap "broker-config" created
    service "broker" created
    statefulset "kafka" created
    Error from server (AlreadyExists): error when creating "kafka\\00namespace.yml": namespaces "kafka" already exists

ここでもPodのデプロイが完了するまで1分程度かかるので、少し待ちます。`kubectl get pod`の結果が以下のようになれば、完了です。

    > kubectl get pod
    > NAME      READY     STATUS    RESTARTS   AGE 
    > kafka-0   1/1       Running   0          1m  
    > kafka-1   1/1       Running   0          1m  
    > kafka-2   1/1       Running   0          41s 
    > kafka-3   1/1       Running   0          28s 
    > pzoo-0    1/1       Running   0          12m 
    > pzoo-1    1/1       Running   0          12m 
    > pzoo-2    1/1       Running   0          11m 
    > zoo-0     1/1       Running   0          12m 
    > zoo-1     1/1       Running   0          12m 

brokerのPodのログを参照して、割り当てられたアドレスを確認してみます。

    > kubectl logs kafka-0 | grep "Registered broker"
    # INFO Registered broker 0 at path /brokers/ids/0 with addresses: PLAINTEXT -> EndPoint(kafka-0.broker.kafka.svc.cluster.local,9092,PLAINTEXT)

"kafka-0.broker.kafka.svc.cluster.local"となっていますが、".svc.cluster.local"の部分は省略可能です。

kube-dnsに対してnslookupして、brokerが名前解決できることを確認してみましょう。クラスター内にbusyboxのPodをデプロイして、そのPodからnslookupを実行します。

    > kubectl run -i --tty --image busybox dns-test --restart=Never --rm /bin/sh
    / # nslookup kafka-0.broker.kafka
    Server:    10.100.0.10
    Address 1: 10.100.0.10 kube-dns.kube-system.svc.cluster.local

    Name:      kafka-0.broker.kafka
    Address 1: 10.244.28.5 kafka-0.broker.kafka.svc.cluster.local

最後に、`exit`を実行してbusyboxのコンソールから抜けておきます。

以上で、Kafkaクラスターのデプロイは完了です。


2 . Kafkaクラスターの動作確認
-----------------------------
ここでは、Kafkaのコマンドライン・ユーティリティを利用して、Kubernetes上にデプロイしたKafkaクラスターの動作を確認します。

### 2-1. 動作確認用のKafkaクライアントのデプロイ
まず、クラスター内にKafkaのクライアントとして利用するPodをデプロイします。

このPodのためのmanifestファイルは、`./kafka/client/kafka-client.yml`です。

    > kubectl create -f ./kafka/client/kafka-client.yml
    pod "kafka-client" created

このPodから、コマンドライン・ユーティリティを実行します。

### 2-2. コマンドライン・ユーティリティを利用したKafkaの動作確認
まず、kafka-clientのコンソールに入ります。

    > kubectl exec -it kafka-client bash

次に、"chatroom"という名前のTopicを作成します。Topicはメッセージを格納するキューに相当する概念です。

    root@kafka-client:/opt/kafka# ./bin/kafka-topics.sh --create --zookeeper zookeeper.kafka:2181 --replication-factor 4 --partitions 12 --topic chatroom
    Created topic "chatroom".

ここでは、`--replication-factor 4 --partitions 12`というオプションを指定しています。これはキューを12個並列に保持(Partition)し、各Partitionのレプリカを3つ、brokerをまたがって分散して保持することを意味します。

以下のコマンドを実行するとTopic "chatroom"の各Partitionが、どのように配置されているかを確認することができます。

    root@kafka-client:/opt/kafka# ./bin/kafka-topics.sh --describe --zookeeper zookeeper.kafka:2181 --topic chatroom
    Topic:chatroom  PartitionCount:12       ReplicationFactor:4     Configs:
            Topic: chatroom Partition: 0    Leader: 1       Replicas: 1,2,3,0       Isr: 1,2,3,0
            Topic: chatroom Partition: 1    Leader: 2       Replicas: 2,3,0,1       Isr: 2,3,0,1
            Topic: chatroom Partition: 2    Leader: 3       Replicas: 3,0,1,2       Isr: 3,0,1,2
            Topic: chatroom Partition: 3    Leader: 0       Replicas: 0,1,2,3       Isr: 0,1,2,3
            Topic: chatroom Partition: 4    Leader: 1       Replicas: 1,3,0,2       Isr: 1,3,0,2
            Topic: chatroom Partition: 5    Leader: 2       Replicas: 2,0,1,3       Isr: 2,0,1,3
            Topic: chatroom Partition: 6    Leader: 3       Replicas: 3,1,2,0       Isr: 3,1,2,0
            Topic: chatroom Partition: 7    Leader: 0       Replicas: 0,2,3,1       Isr: 0,2,3,1
            Topic: chatroom Partition: 8    Leader: 1       Replicas: 1,0,2,3       Isr: 1,0,2,3
            Topic: chatroom Partition: 9    Leader: 2       Replicas: 2,1,3,0       Isr: 2,1,3,0
            Topic: chatroom Partition: 10   Leader: 3       Replicas: 3,2,0,1       Isr: 3,2,0,1
            Topic: chatroom Partition: 11   Leader: 0       Replicas: 0,3,1,2       Isr: 0,3,1,2

次に、KafkaのConsumer（メッセージを取得するクライアント）を起動します。以下のコマンドを実行すると待機状態となり、brokerにメッセージが投入されるのを待ち受けている状態になります。

    root@kafka-client:/opt/kafka# ./bin/kafka-console-consumer.sh --bootstrap-server kafka-0.broker.kafka:9092 --topic chatroom --from-beginning

続いて、KafkaｎProducer（メッセージを登録するクライアント）を起動します。新たにターミナル／PowerShellを起動し、kafka-clientのコンソールに入ります。

    > kubectl exec -it kafka-client bash

Produerを起動します。以下のコマンドを実行すると、コンソールが入力待ちの状態となりますので、適当な文字列を入力したあと`Return`キーを打ちます。

    root@kafka-client:/opt/kafka# ./bin/kafka-console-producer.sh --broker-list kafka-0.broker.kafka:9092,kafka-1.broker.kafka:9092,kafka-2.broker.kafka:9092,kafka-3.broker.kafka:9092 --topic chatroom
    >hoge
    >bar
    
Consumerを起動した側のコンソールに、Producerで入力したメッセージが表示されることを確認してください。

動作が確認できたら、Consumer、Producerを[Ctrl + C]で終了し、それぞれ`exit`を入力して、kafka-clientのコンソールを抜けておきます。

_（以降の手順では、特に断りのない限り最初に立ち上げた方のコンソールで作業を行います）_


3 . チャットアプリケーションを動かしてみる
------------------------------------------
ここまでの手順で構成したKafkaクラスターを利用して、チャットアプリケーションを動かしてみます。

クラスター内には、以下の2つのサーバーを追加でデプロイしておきます。

- メッセージを受け取ってKafkaにPublishするREST APIサーバー（以下、Publishサーバー）
- メッセージをSubscribeしてクライアントに送るServer Sent Event(SSE)のサーバー（以下、Subscribeサーバー）

チャットアプリケーションのクライアントは、Kubernetesクラスターの外で動かし、REST API、SSEを使ってメッセージをやり取りします。

<!-- 図 -->

Publish/Subscribeサーバーおよびチャットクライアントのコードは[GitHubにあります](https://github.com/hhiroshell/kchat)ので、ご興味ある方は参照してみてください。


### 3-1. Publish/Subscribeサーバーのデプロイ
Publish/SubscribeサーバーはそれぞれDockerコンテナ化してDocker Hubにアップロードしてあります。

`./kchat/`配下のmanifestを使って、Publish/SubScribeサーバーをデプロイします。

    > kubectl create -f ./kchat/
    service "kchat-server-kconsumer" created
    deployment "kchat-server-kconsumer" created
    service "kchat-server-kproducer" created
    deployment "kchat-server-kproducer" created


### 3-2. チャットクライアントの実行
チャットクライアントはJavaで実装されたアプリケーションです。手元の環境にJRE 1.8+がインストールされていない場合は、ここでインストールをしてください（一時借出し環境にはインストール済みです）。

インストールが完了したら、カレントディレクトリをこのリポジトリの最上位のフォルダ(cndjp3)に変更します。

    > cd ../

まず、チャットクライアントのバイナリをダウンロードします。

・Mac/Linux

    > wget https://github.com/hhiroshell/kchat/releases/download/v0.1/kchat.client-jar-with-dependencies.jar

・Windows

    > Invoke-WebRequest -Uri https://github.com/hhiroshell/kchat/releases/download/v0.1/kchat.client-jar-with-dependencies.jar -OutFile kchat.client-jar-with-dependencies.jar

続いて、チャットクライアントを実行します。正常に起動すると、メッセージの入力／受信待ち状態になります。

    > java -jar kchat.client-jar-with-dependencies.jar 172.17.8.102 "User A"
    User A: 

新たにターミナル／PowerShellを起動し、同様にチャットクライアントを起動します。

    > java -jar kchat.client-jar-with-dependencies.jar 172.17.8.102 "User B"
    User B: 

一方のコンソールで文字列を入力し`Return`すると、一方のコンソールにもそれが表示されることを確認してください。

チャットクライアントを終了するには`Ctrl + C`を押下します。


以上で、ハンズオンのPart1は終了です。
