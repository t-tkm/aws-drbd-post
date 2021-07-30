# 1.はじめに
本記事では、クラウドでのDRBDを用いたディスクミラーリング方式についての読込み・書込み性能について検証します。

DRBDによりミラーリングされたディスクでは、複数のディスクにデータを書込むためオーバーヘッドが発生します(*)。特に、(AZ横断など)データセンタを跨いだ場合には、物理的な距離が遠くなるため、書込み性能への影響が気になり、その程度を測ってみることにしました。

(*)読込み性能については、どこか1箇所を参照するだけなのでオーバーヘッドはありません。

尚、記事内には参考の為、生データもダウンロードできるようにしていますが、当記事の目的は、各種性能測定ツールを用いたやり方の情報提供になります。測定は、手元で簡易的に動かして得られたものです。測定結果の正確性、有用性などにつきましては、一切保証するものではない点ご了承ください。

# 2. クラウドでの共有ディスク代替方式
オンプレでのHAクラスタをクラウド上で実現する際、単純には共有ディスク構成を作ることができません(2020.8)。(AZやゾーンといった)**データセンタ内**であれば、AWSではマルチアタッチを使用して複数インスタンスにEBSを割当てたり、Azureでも同様な機能は提供されてはいます。しかし、いずれの共有ディスクサービスにも制約はあるため、要件を満たせるかの確認が必要になります。

- [Amazon EBSのマルチアタッチ機能とAzure Shared Disksを比較してみる](https://qiita.com/hayao_k/items/b978cd26622536000a6a)
- [EBS Multi-Attachの使いどころを考えてみた](https://dev.classmethod.jp/articles/ebs-multi-attach-usecase-thought/)

<div align="center">
  <a href="https://github.com/t-tkm/aws-drbd-post/blob/images/imgs/img1.png?raw=true">
    <img alt="img1" src="https://github.com/t-tkm/aws-drbd-post/blob/images/imgs/img1.png?raw=true">
  </a>
</div>

一方、共有ディスクの代替として、ノード間でデータをリアルタイムに同期させて共有するミラーリング方式や、リアルタイムではなく、一定間隔でデータの同期をとるレプリケーション方式といった他のやり方もあります。その他、Cephなど分散ストレージを使うというのも一手かもしれません。

それぞれの方式にはメリット・デメリットがありますが、ここではオンプレでの共有ディスクを用いたHA構成をクラウドにもっていく時によく使われる「ミラーリング方式」について検討してみます。

# 3.システム構成
測定は、下記2パターンでの影響を試してみました。

1. [アプリケーション負荷測定]PostgreSQLと、そのベンチマークツールpgbenchを使った計測
2. [ディスク負荷測定]ディスク負荷ツールfioを使った計測

<div align="center">
  <a href="https://github.com/t-tkm/aws-drbd-post/blob/images/imgs/img2.png?raw=true">
    <img alt="img2" src="https://github.com/t-tkm/aws-drbd-post/blob/images/imgs/img2.png?raw=true">
  </a>
</div>

AWSでは、東京リージョンの2つのAZを用いて、下記構成でリソースを準備しました。

<div align="center">
  <a href="https://github.com/t-tkm/aws-drbd-post/blob/images/imgs/img3.png?raw=true">
    <img alt="img3" src="https://github.com/t-tkm/aws-drbd-post/blob/images/imgs/img3.png?raw=true">
  </a>
</div>

各ホストの情報を、参考までに記載しておきます:
- host1-3共通:
    - t2/medium
    - CentOS 8.2.2004
    - Linux version 4.18.0-193.6.3.el8_2.x86_64
    - DRBD 9系

- host1:
    - IP アドレス: 172.16.1.50
    - ディスク:
        - /dev/xvda - OS
        - /dev/xvdb – DRBD (/dev/drbd0)
        - /dev/xvdc – DRBD (/dev/drbd1) 
        - /dev/xvdd – ローカル
- host2:
    - IP アドレス: 172.16.1.60
    - ディスク:
        - /dev/xvda - OS
        - /dev/xvdb – DRBD (/dev/drbd0) 
- host3:
    - IP アドレス: 172.16.2.50
    - ディスク:
        - /dev/xvda - OS
        - /dev/xvdb – DRBD (/dev/drbd1) 

すこしみやすく、図にしてみると、このようになります。

<div align="center">
  <a href="https://github.com/t-tkm/aws-drbd-post/blob/images/imgs/img4.png?raw=true">
    <img alt="img4" src="https://github.com/t-tkm/aws-drbd-post/blob/images/imgs/img4.png?raw=true">
  </a>
</div>

# 4.環境構築
この章では、測定に必要となるOSやミドルウェアをインストール、セットアップしていきます。初めに、CentOS 8.2.2004のパッケージ管理ツールdnfを用いて、DRBD用リポジトリの追加や、プロファイリングツールのインストールなど準備を行います。

```
# dnf update
# rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
# dnf install https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm
# dnf install -y sysstat
# dnf install -y dstat
# dnf install -y fio
```
## 4-1. DRBDのインストール

DRBDをインストール、セットアップします。drbd90パッケージをインストールしました。

```
# dnf install -y kmod-drbd90
# dnf install -y drbd90
```

無事にDRBDがインストールされれば、DRBDの設定に移ります。host1-host2間の同期設定を行います。host1で作成し、host2へコピーします。

```:/etc/drbd.d/r0.res
resource r0{
  net {
   protocol C;
  }
  options {
  }
  on ip-172-16-1-50.ap-northeast-1.compute.internal {
    address 172.16.1.50:7789;
    device /dev/drbd0;
    disk /dev/xvdb;
    meta-disk internal;
  }
  on ip-172-16-1-60.ap-northeast-1.compute.internal {
    address 172.16.1.60:7789;
    device /dev/drbd0;
    disk /dev/xvdb;
    meta-disk internal;
  }
}
```

host1-host3間の同期設定。host1で作成し、host3へコピーします。

```:/etc/drbd.d/r1.res
resource r1{
  net {
   protocol C;
  }
  options {
  }
  on ip-172-16-1-50.ap-northeast-1.compute.internal {
    address 172.16.1.50:7790;
    device /dev/drbd1;
    disk /dev/xvdc;
    meta-disk internal;
  }
  on ip-172-16-2-50.ap-northeast-1.compute.internal {
    address 172.16.2.50:7790;
    device /dev/drbd1;
    disk /dev/xvdb;
    meta-disk internal;
  }
}
```

host1-host2間のリソース作成と有効化。host1, 2で実行します。

```
# drbdadm create-md r0
# drbdadm up r0
```

host1-host3間のリソース作成と有効化。host1, 3で実行します。

```
# drbdadm create-md r1
# drbdadm up r1
```

下記はhost1のみで実施します。

```
// host1-host2間同期開始
# drbdadm primary --force r0
// host1-host3間同期開始
# drbdadm primary --force r1

// 同期中
# drbdadm status
r0 role:Primary
  disk:UpToDate
  ip-172-16-1-60.ap-northeast-1.compute.internal role:Secondary
    replication:SyncSource peer-disk:Inconsistent done:83.70

r1 role:Primary
  disk:UpToDate
  ip-172-16-2-50.ap-northeast-1.compute.internal role:Secondary
    replication:SyncSource peer-disk:Inconsistent done:73.31

// 同期完了
# drbdadm status
r0 role:Primary
  disk:UpToDate
  ip-172-16-1-60.ap-northeast-1.compute.internal role:Secondary
    peer-disk:UpToDate

r1 role:Primary
  disk:UpToDate
  ip-172-16-2-50.ap-northeast-1.compute.internal role:Secondary
    peer-disk:UpToDate
```

最後に、ファイルシステムを作成し、適当なディレクトリにマウントして置きます。

```
// ファイルスシステム作成
mkfs -t xfs /dev/drbd0
mkfs -t xfs /dev/drbd1
mkfs -t xfs /dev/xvdd

// マウント
mount /dev/drbd0 /mnt/drbd0-fs/
mount /dev/drbd1 /mnt/drbd1-fs/
mount /dev/xvdd /mnt/local-fs/

// 確認
# df -h
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        1.9G     0  1.9G   0% /dev
tmpfs           1.9G  8.0K  1.9G   1% /dev/shm
tmpfs           1.9G   17M  1.9G   1% /run
tmpfs           1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/xvda2      8.0G  1.3G  6.7G  17% /
tmpfs           378M     0  378M   0% /run/user/1000
/dev/xvdd       100G  858M  100G   1% /mnt/local-fs
/dev/drbd0      100G  858M  100G   1% /mnt/drbd0-fs
/dev/drbd1      100G  826M  100G   1% /mnt/drbd1-fs
```

## 4-2.PostgreSQLのインストール
PostgreSQLは、CentOS標準リポジトリから、下記でインストールしました。今回は性能測定にpgbenchというツールを使うため、合せてそちらもインストールしておきます。

```
# dnf install -y postgresql
# dnf install -y postgresql-server
# dnf install -y postgresql-contrib  //pgbench用
```

次に、pgbenchを使うためのデータベースの設定を行います。今回は、1.AZ内レプリケーション(DRBD0)、2.AZ跨ぎレプリケーション(DRBD1)、3.ローカルディスクの3パターンで測定を実施するため、データベースは下記マウントしたファイルシステムをそれぞれ利用します。DRBDのセットアップ時に作成した、/mnt以下のディレクトリを確認します。

```
// mountポイント確認
# cd /mnt
//postgreSQLは、デフォルトpostgresユーザを利用するため、所有者とパーミッションを下記の通り変更します。
# chown -R postgres:postgres drbd0-fs drbd1-fs local-fs
# chmod -R 700 drbd0-fs drbd1-fs local-fs
// 結果はこんな感じになります
# ll
total 0
drwxr-xr-x 2 postgres postgres 6 Jul 26 04:06 drbd0-fs
drwxr-xr-x 2 postgres postgres 6 Jul 26 04:06 drbd1-fs
drwxr-xr-x 2 postgres postgres 6 Jul 26 04:08 local-fs
```

3つのデータベースは、今回はSystemdの設定を使って切替える事にしました(もっとうまい方法があるかと思いますが、今回は簡単にディレクトリ事データベースを取替える方式を採用...)。下記のようにUnit定義の中の「PGDATA」として3つ準備します。

```:/usr/lib/systemd/system/postgresql.service 
(中略)
#Environment=PGDATA=/var/lib/pgsql/data
Environment=PGDATA=/mnt/drbd0-fs 
#Environment=PGDATA=/mnt/drbd1-fs
#Environment=PGDATA=/mnt/local-fs
(中略)
```

その後、上記各設定ファイルのコメントアウトを(1つずつ)削除し、下記コマンドを実行しデータベースの初期化を行います(3回実施することになります)。

```
# systemctl daemon-reload   #Unit定義更新
# postgresql-setup initdb   #データベース初期化
```

データベースのセットアップは以上になります。

# 5.アプリ性能測定(pgbench)
## 5-1.測定方法
以下、pgbenchによる測定を行います。

- pgbenchマニュアル (https://www.postgresql.jp/document/9.2/html/pgbench.html)

このツールの特徴としましては、マニュアルによるとTCP-B(銀行のバッチ処理)を模したベンチマークとのことです。
>pgbenchはPostgreSQL上でベンチマーク試験を行う単純なプログラムです。 これは同一のSQLコマンドの並びを何度も実行します。複数の同時実行データベースセッションで実行することもできます。 そして、トランザクションの速度（1秒当たりのトランザクション数）の平均を計算します。 デフォルトでpgbenchは、1トランザクション当たり5つのSELECT、UPDATE、INSERTコマンドを含むおおよそTPC-Bに基いたシナリオを試験します

```
# systemctl daemon-reload   #使うデータベース格納場所(PGDATA)をコメントアウトし、Unit定義を更新
# systemctl start postgresql.service   # データベース起動
# sudo su - postgres
$ createdb test   #testという名のデータベース作成
$ pgbench -i test   #pgbench用のテストデータをtestデータベースへロード
$ pgbench -c 10 -t 1000 -l test   #測定実行
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
number of transactions per client: 1000
number of transactions actually processed: 10000/10000
latency average = 9.111 ms
tps = 1097.529558 (including connections establishing)
tps = 1097.770942 (excluding connections establishing)
$ exit   #postgresユーザから抜ける
# systemctl start postgresql.service   #データベース停止
```

上記より性能(TPS)は1097程度出ている事がわかります(PostgreSQLへの接続オーバーヘッド有り/無しで2パターンTPSは表示さる)。また、オプションとして「-l」を指定している事により実行後ローカルに「pgbench_log.1163」というトランザクション詳細結果ファイルが作成されます(詳細につきましては、上記サイトを確認ください)。これをデータベースを切替えてそれぞれ実行します。

## 5-2.測定結果
ざっくり、共有ディスク方式と比較した場合の処理性能(TPS)は、AZ内で62.5%(37.5%減)、AZ跨ぎで26.9%(73.1%減)という結果となりました。

<div align="center">
  <a href="https://github.com/t-tkm/aws-drbd-post/blob/images/imgs/img5.png?raw=true">
    <img alt="img5" src="https://github.com/t-tkm/aws-drbd-post/blob/images/imgs/img5.png?raw=true">
  </a>
</div>

# 6.ディスク性能測定(fio)
## 6-1.測定方法
ディスク性能測定に使えるベンチマークツールとしては、[詳解 システム・パフォーマンス](https://www.amazon.co.jp/dp/4873117909)によると、dd、Bonnie、Bonnie++、iozone、tiobench、SysBench、fio、FileBenchなど多々ありますが、今回はfio(Flexible IO Tester)を使ってみました(著者のJens Axboe氏は同年代だそうで少し親しみもあります)。高機能なのですが、全く使いこなせていないため、今後勉強していきたいと思います！

ジョブファイルという設定ファイルを設定し、カレントディレクトリに格納します。
※各種詳細は[こちら](https://fio.readthedocs.io/en/latest/index.html)。

以下が、fio設定ファイル(サンプル)となります:
```:fio.conf
[global]
ioengine=libaio
iodepth=1
size=1g
direct=1
runtime=30
stonewall

[Seq-Write-1M]
bs=1m
rw=write

[Rand-Write-1M]
bs=1m
rw=randwrite

[Rand-Write-512K]
bs=512k
rw=randwrite

[Rand-Write-4K]
bs=4k
rw=randwrite
```

次に、負荷をかけている間のプロファイリングには「iostat」と「dstat」というツールを利用しました。Terminalを3つ立上げ、先にiostatとdstatを起動した上でfioを実行。fioの実行が終わったら、iostatとdstatをCtl+Cで止めます。fioの実行では、「--filename」オプションではディレクトリだけでなくデバイスも指定できるようです。

```
//Terminal1で実行
iostat -mx 1 | grep --line-buffered drbd0 |  awk '{print strftime("%y/%m/%d %H:%M:%S"), $0}' | tee iostat_drbd0.txt
//Terminal2で実行
dstat -t | tee dstat_drbd0.txt
//Terminal3で実行
fio --filename=/dev/drbd0 fio.conf
```

これを、drbd0、drbd1、ローカルディスクそれぞれで実行してプロファイルした結果は次の通りです。

## 6-2.測定結果(1)(スループット)
下記がiostatでの集計結果になります。ブロックサイズとして1M、512K、4Kとバリエーション指定してみたのですが、残念ながらEC2インスタンスのネットワーク帯域の制限がかかってしまっているようで、1Mと512Kのブロックサイズ指定は有効ではなかったようです(なんとなく2.mediumタイプを選んでしまったことが敗因)。また、EBS自体もボリュームサイズによりIOPS制約が異なるため、次回は測定条件の見直しから必要だと反省しました。

4Kブロックの結果より、ローカルが性能が一番よく、続いてAZ内、AZ跨ぎの順番が確認できました。

<div align="center">
  <a href="https://github.com/t-tkm/aws-drbd-post/blob/images/imgs/img6.png?raw=true">
    <img alt="img6" src="https://github.com/t-tkm/aws-drbd-post/blob/images/imgs/img6.png?raw=true">
  </a>
</div>

DRBD0のRAWデータの一部抜粋↓
<div align="center">
  <a href="https://github.com/t-tkm/aws-drbd-post/blob/images/imgs/img7.png?raw=true">
    <img alt="img7" src="https://github.com/t-tkm/aws-drbd-post/blob/images/imgs/img7.png?raw=true">
  </a>
</div>

## 6-3.測定結果(2)(write待ち時間)
iostatの「w_await」結果はこのようになりました。

w_awaitの定義は、[iostat manual](https://man7.org/linux/man-pages/man1/iostat.1.html)↓になります。
>The average time (in milliseconds) for write requests issued to the device to be served. This includes the time spent by the requests in queue and the time spent servicing them.

<div align="center">
  <a href="https://github.com/t-tkm/aws-drbd-post/blob/images/imgs/img8.png?raw=true">
    <img alt="img8" src="https://github.com/t-tkm/aws-drbd-post/blob/images/imgs/img8.png?raw=true">
  </a>
</div>

# 7.まとめ
共有ディスク方式をDRBDを用いたミラーリング方式に変更し、AWS上に単純リフトした場合の性能を少し実験してみました。結果につきましては、要件次第で「この程度なら問題ないね」や、「ちょっと厳しいかも」など変わるかと思います。また、アーキテクチャ見直しや（クラウドであればもう少しクラウドネイティブな等）、様々なレイヤにおけるチューニング方法もあり([RAIDを使うなど](https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/raid-config.html))、これをもって一概に「性能が悪くなる」と言えないところがエンジニアリングの楽しみかと思います。ではどうすれば良いのか、についきましては追々勉強していきたいと思います。

※測定データは[こちら]()からダウンロードできます。

# 参考
- [詳解 システム・パフォーマンス](https://www.amazon.co.jp/dp/4873117909)
- [ファイルシステムのベンチマーク集](http://www.nminoru.jp/~nminoru/unix/fs_benchmarks.html)