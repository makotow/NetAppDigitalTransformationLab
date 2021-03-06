=============================================================
インフラの可用性を向上させる
=============================================================


k8s Master の冗長化
=============================================================

API受付をするマスタ系のノードやetcdやkubernetesサービスの高可用性も担保しましょう。

また、障害設計をどの単位(DC単位、リージョン単位）で行うかも検討をしましょう。

* https://kubernetes.io/docs/tasks/administer-cluster/highly-available-master/
* https://github.com/kubernetes/community/blob/master/contributors/design-proposals/cluster-lifecycle/ha_master.md


ログの確認、デバッグ方法
=============================================================

標準のkubectlだとログが追いづらいときがあるため以下のツールの検討をします。

* kubernetesホストの/var/log/containerにログは保管。(systemd系の場合）
* sternなどのログ管理ツールを活用する
* fluentdを使いログを集約する。

コンテナクラスタの監視
=============================================================

監視する対象として、メトリクス監視、サービス呼び出し、サービスメッシュなど分散環境になったことで従来型のアーキテクチャとは違った監視をする必要があります。
簡単にスケールアウトできる＝監視対象が動的というような考え方をします。

また、分散環境では１つのアプリケーションでも複数のサービス呼び出しがおこなわれるため、どのようなサービス呼び出しが行われているかも確認できる必要があります。

* (heapster|Prometheus) + Grafana + InfluxDB を使いコンテナクラスタの監視。
* 分散環境に於けるサービスの呼び出しを可視化 Traces = Zipkin
* ServiceGraph Graphbiz & Prometeus
* ServiceMesh

Helmで提供されているGrafana+Prometheusをデプロイし監視することにチャレンジしてみましょう。

:doc:`./prometheus/deploy-monitoring`

.. todo:: Prometheusの長期保管について記載する。

Prometheusは長期保管はできないため長期保管の際は別途保管が必要です。

バックアップはどうするか？
=============================================================

大きく以下のバックアップ対象が存在します。

* etcd
* コンテナ化されたアプリケーションの永続化データ
* コンテナイメージ

etcd のバックアップ戦略
----------------------------------------------------------------

kubernetes の構成情報が保存されているetcdのバックアップをどうするかについてわかりやすいドキュメントが公開されています。
基本方針としては、etcdctlのスナップショットを定期的にとる、またはストレージ機能でとるという２つの大きな方針です。

- https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/recovery.md

参考までに etcdctl を使ったサンプルを提示します。

    .. code-block:: console

        ETCDCTL_API=3 etcdctl --debug --endpoints https://ip_address:2379 --cert="server.crt" --key="server.key" --cacert="ca.crt" snapshot save backup.db

この実行内容をCronJob等で定期的に取得し保管するという方法が取れます。

コンテナ化されたアプリケーションの永続化データ
----------------------------------------------------------------

基本方針として永続化するデータは外部ストレージに保管するという方針でいくと、
ストレージ側でバックアップを取得するのが比較的容易にバックアップ可能です。

ご参考までに、trident ではストレージスナップショットを定期的に取得するよう設定可能です。
1サイトでのバックアップは簡易的に可能ですが、遠隔地保管等をする場合は後述の「DRをどうするか？」で言及します。

コンテナイメージ
----------------------------------------------------------------

上記２つとは考え方が違うものになります。
クラウドのサービスを使う上では可用性や冗長性はSLAに従うことになり、ユーザ側で意識することはあまりありません。
プライベートレジストリを利用する場合は、ダウンしてしまうと新たにアプリケーションがデプロイできないという自体になってしまいます。

セキュリティアップグレード
=============================================================

例えば、脆弱性があった場合の対処方法はどうすればよいか。

* ノードごとにバージョンアップするため、ある程度の余力を見込んだ設計とする。
* kubectl drain を使いノードで動いているPodを別ノードで起動、対象ノードをアップデートし、ポッドを戻す。

DRをどうするか？
=============================================================

アプリケーションのポータビリティはコンテナで実現。
別クラスタで作成されたPVはそのままは参照できないので以下の方法を検討する。

* CSI (Container Storage Interface)の既存ボリュームのインポートに対応をまつ
* Heptio ark: https://github.com/heptio/ark + SVM-DR

..
    Heptio ark を使ったDRについては以下の章でまとめました。

    .. include:: ./ark-svmdr/dr.rst


Podが停止した場合のアプリケーションの動作
=============================================================

Dynamic ProvisioningされたPVCのPod障害時の動作については以下のような動作になります。
PVCはTridentを使ってデプロイしたものです。

- 停止したPodは別ノードで立ち上がる
- Podからマウントしていたボリュームは再度別ノードでもマウントされデータの読み書きは継続可能

``Stateful Set`` を使い、MongoDBを複数ノードで構成し上記の検証を行った結果が以下のリンク先で確認できます。

:doc:`./mongodb-statefulset-failure/statefulset-failure`
