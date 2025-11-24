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
1. `maxUnavailable`: 最大いくつのPodを同時にシャットダウンできるか。e.g. 25%だったら4つPodがある場合、1つずつPodを再作成する。パーセントではなく個数を書くことも可能。
1. `maxSurge`: 最大いくつのPodを新規作成できるか。

これらを適切に設定しておくことでPodの数が増えすぎるのを防ぐことができる。

Podの数が増えすぎると以下の様な困りが起きうる。

- インフラコストが増大する
- ノードのキャパシティが枯渇してしまい、Rolling Updateが終わらなくなる
