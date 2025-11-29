# bbf-kubernetes
つくって、壊して、直して学ぶ Kubernetes入門 https://www.shoeisha.co.jp/book/detail/9784798183961

公式リソース: https://github.com/aoi1/bbf-kubernetes

## Chapter 2 Kubernetesクラスタをつくってみる
- Control PlaneがWoker Noteに直接指示するのではなく、Woker NodeがControl Planeに問い合わせるアーキテクチャになっている。(コレオグラフィっぽい感じ？)

## Chapter 4 アプリケーションをKubernetesクラスタ上につくる
### `Namespace`
- Namespaceは単一クラスタ内のリソース群を分離するメカニズムを提供する。

## Chapter 5 トラブルシューティングガイドとkubectlコマンドの使い方
### `kubectl debug`
- デバッグ用のサイドカーコンテナを立ち上げるコマンド
- distrolessなど、シェルを同梱していないコンテナイメージを使っている場合などに便利そう。


### `kubectl run`
- コンテナを即時に実行する
```sh
$ kubectl debug --stdin --tty myapp --image=curlimages/curl:8.4.
0 --target=hello-server -- sh
```

```sh
$ kubectl run  busybox --image=busybox:1.36.1 --rm --stdin --tty --restart=Never --command -- nslookup google.com
```

### `kubectl exec`
- コンテナにログインする
```sh
$ kubectl get pod myapp --output wide                           
NAME    READY   STATUS    RESTARTS   AGE    IP           NODE                 NOMINATED NODE   READINESS GATES
myapp   1/1     Running   0          127m   10.244.0.5   kind-control-plane   <none>           <none>

$ kubectl exec --stdin --tty curlpod -- /bin/sh

# コンテナ内
~ $ curl 10.244.0.5:8080
```

### `kubectl port-forward`
- port-forwardでアプリケーションにアクセス
    - PodにはKubernetesクラスタ内用のIPアドレスが割り当てられるため、何もしないとクラスタ外からアクセスできない
```sh
# myappの8080番ポートをホストマシンの5555番ポートにポートフォワード
$ kubectl port-forward myapp 5555:8080

# 別のシェルから
$ curl localhost:5555
Hello, World!
```

### `kubectl edit`
マニフェストをその場で編集する

```sh
$ kubectl edit pod myapp
```


### `kubectl delete`
- リソースを削除する
- kubectlには「Podを再起動する」というコマンドがないため`kubectl delete`コマンドで代替する

```sh
$ kubectl delete pod myapp
```

## Chapter 6 Kubernetes リソースをつくって壊そう
- ReplicaSetとDeployment
    - DeploymentはReplicaSetというリソースを作り、ReplicaSetがPodを作成する。
    - Deployment -> ReplicaSet -> Pod という関係性

### `ReplicaSet`
> ReplicaSetは指定した数のPodを複製するリソースです。Podリソースと異なるところは、Podを複製できるところです。複製するPodの数をreplicasで指定できます。

### `Deployment`
本番環境で無停止でコンテナイメージのアップデートを行うためには複数のReplicaSetが必要になる。ReplicaSetを管理する上位概念がDeployment。

### `StrategyType`
Deploymentを利用してPodを更新するときに、どのような戦略で更新するかを指定する設定値。

以下の2つが選択可能。
1. `Recreate`: 全部のPodを同時に更新する
1. `RollingUpdate`: Podを順番に更新する。別途`RollingUpdateStrategy`を記載することができる。

`RollingUpdateStrategy`に指定できる値は以下の2つ。
1. `maxUnavailable`: 最大いくつのPodを同時にシャットダウンできるか。e.g. 25%だったら4つPodがある場合、1つずつPodを再作成する。パーセンテージまたは整数で指定できる。パーセンテージを指定した場合、*絶対値は少数切り下げで計算される。*
1. `maxSurge`: 理想状態のPod数を超えて作成できる最大のPod数。パーセンテージまたは整数で指定できる。パーセンテージを指定した場合、*絶対値は少数切り上げで計算される。*

これらを適切に設定しておくことでPodの数が増えすぎるのを防ぐことができる。

Podの数が増えすぎると以下の様な困りが起きうる。

- インフラコストが増大する
- ノードのキャパシティが枯渇してしまい、Rolling Updateが終わらなくなる

### `kubectl delete`でDeploymentのpodを削除してみる
`kubectl delete pod <pod_name>` でpodを削除しても一瞬で新しいpodが立ち上がる。

### `Service`
- DeploymentはIPアドレスを持たないため、Deploymentで作成したPodにアクセスするためには個々のPodのIPアドレスを指定して接続する必要がある。
- Serviceを用いると`service-name.default.svc.cluster.local`のようなドメイン名でアクセスできるようになる。

#### `Service`のType
- `ClusterIP` (デフォルト): クラスタ内部のIPアドレスでServiceを公開する。このTypeで指定されたIPアドレスはクラスタ内部からしかアクセスできない。`Ingress`というリソースを利用することで外部公開が可能になる。
- `NodePort`: すべてのNodeのIPアドレスで指定したポート番号を公開する
- `LoadBalancer`: 外部ロードバランサを用いて外部IPアドレスを公開する。LBは別途用意する必要がある。
- `ExternalName`: ServiceをexternalNameフィールドの内容にマッピングする。(e.g. `api.example.com`)このマッピングにより、クラスタのDNSサーバがその外部ホスト名の値をもつCNAMEレコードを返すように設定される。

### `ConfigMap`
- 環境変数など、コンテナの外部から値を設定したいときに利用するリソース。

ConfigMapを利用する方法は3つある。
1. コンテナ内のコマンドの引数として読み込む
1. コンテナの環境変数として読み込む
1. ボリュームを利用してアプリケーションのファイルとして読み込む

### `Secret`
- ConfigMapに機密情報を登録してしまうと、誰でもその情報(e.g. DBへの接続情報)を参照できてしまう。
- Secretというリソースを用いるとアクセス権分けることができる。
    - Secretのデータはbase64でエンコードして登録する必要がある。

SecretをPodに読み込む方法は以下の2つ。

1. コンテナの環境変数として読み込む
1. ボリュームとしてコンテナに設定ファイルを読み込む

### `Job`
- Jobは一つ以上のPodを作成し、指定された数のPodが正常に終了するまでPodの実行を再試行し続ける。
- Podが正常に終了するとJobは成功したPodの数を追跡する。指定された完了数に達するとそのJobは完了したとみなされる。
- Jobを削除すると作成されたPodも一緒に削除される。
- Jobを一時停止すると再開されるまで、稼働しているPodは全部削除される。

ref: https://kubernetes.io/ja/docs/concepts/workloads/controllers/job/

### `CronJob`
- CronJobは定期的にJobを生成するリソース。
- CronJobはJobを作成し、JobはPodを作成する。

