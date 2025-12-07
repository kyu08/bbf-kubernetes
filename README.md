つくって、壊して、直して学ぶ Kubernetes入門 https://www.shoeisha.co.jp/book/detail/9784798183961

公式リソース: https://github.com/aoi1/bbf-kubernetes

## Chapter 2 Kubernetesクラスタをつくってみる
- Control PlaneがWorker Noteに直接指示するのではなく、Worker NodeがControl Planeに問い合わせるアーキテクチャになっている。(コレオグラフィっぽい感じ？)

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

#### `StrategyType`
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

## Chapter 7 安全なステートレス・アプリケーションをつくるには
まずはじめに3種類のProbeについて説明する。

- Readiness probe
- Liveness probe
- Startup probe

### `Readiness probe`
- コンテナがReadyになるまでの時間やエンドポイントを制御するのがReadiness probe。
- Readiness probeが失敗すると、Serviceリソースの接続対象から外され、トラフィックを受けなくなる。

### `Liveness probe`
- Liveness probeが失敗するとPodが再起動される。
- 再起動で治るケースでは問題ないが、最悪の場合再起動の無限ループに入る可能性があるので注意が必要。
- Readiness probeを待つような挙動が組み込まれているわけではないため、`initialDelaySeconds`を調整するかStartup probeを使用する必要がある。

### `Startup probe`
- Startup probeはコンテナの初回起動時にのみ利用するProbe。
- Startup probeが成功するまで、Liveness probeとReadiness probeは無視される。
- [Kubernetes version 1.18](https://github.com/kubernetes/kubernetes/releases/tag/v1.18.0)から導入された機能。(Mar 26, 2020にリリースされたバージョンらしい。)

マニフェストは以下のような形式。

```yaml
startupProbe:
    httpGet:
        path: /healthz
        port: liveness-port
    failureThreshold: 30
    periodSeconds: 10
```

- `failureThreshold`: Probeが失敗した場合、KubernetesはfailureThresholdに設定した回数までProbeを試行します。 Liveness Probeにおいて、試行回数に到達することはコンテナを再起動することを意味します。 Readiness Probeの場合は、Podが準備できていない状態として通知されます。デフォルトは3。最小値は1。
- `periodSeconds`: Probeが実行される頻度(秒数)。デフォルトは10秒。最小値は1。 コンテナが起動してから準備が整うまでの間、periodSecondsで指定した間隔とは異なるタイミングでReadiness Probeが実行される場合があります。 これは、Podをより早く準備完了の状態に移行させるためです。

ref: https://kubernetes.io/ja/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

### コンテナのリソース使用量を要求する `Resource requests`
- 確保したいリソースの最低使用量を指定することができる設定値。
- Kubernetesのスケジューラはこの値を見てスケジュールするNodeを決定する。
- どのNodeもRequestsにかかれている量が確保できなければPodはスケジュールされない。

### コンテナのリソース使用量を制限する `Resource limits`
- コンテナが使用できるリソース使用量の上限を指定する設定値。
- コンテナはこのLimitsを超えてリソースを使用することはできない。
- メモリが上限値を超える場合、OOMでPodがKillされる。
- CPUが上限値を超えた場合は即座にPodがKillされるのではんくスロットリングが発生し、アプリケーションの動作が遅くなる。

### メモリ
- 単位を指定しない場合、`1`は1byteを意味する。
- K, M, Gなどの接頭語だけでなく、Ki, Mi, Giなども利用できる。

### CPU
- 単位を指定しない場合、`1`はCPUの1コアを意味する。

### PodのQuality of Service(QoS) Classes
- OOM KillerはQoSに応じてOOM KillするPodの優先順位を決定し、必要に応じて優先度の低いPodからOOM Killする。
- QoSクラスには次の3種類がある。


| クラス名 | OOM Killの優先度 | 条件 |
|----------|------------------|------|
| Guaranteed | 3 | Pod内のすべてのコンテナにリソースのrequestsとlimitsが指定されている。さらに、メモリとCPUの両方にrequests=limits、となる値が指定されている。 |
| Burstable | 2 |  Pod内のコンテナのうち少なくとも1つはメモリまたはCPUのrequests/limitsが指定されている。 |
| BestEffort | 1 | GuaranteedでもBurstableでもないもの。リソースに何も指定していない。 |

k9sでdescribeしたときの様子。QoS Classが表示されている。

<img width="4008" height="2484" alt="Image" src="https://github.com/user-attachments/assets/c0f8624a-69ec-4417-ad1d-0d42cad9aab6" />

### Nodeを指定する: `Node selector`
- 特定のNodeにのみスケジュールするという制御を行う機能。

### Podのスケジュールを柔軟に指定する: `Affinity`と`Anti-Affinity`
Affinity, Anti-Affinityには以下の3種類がある。

1. Node affinity
1. Pod affinity
1. Pod anti-affinity

#### `Node affinity`
- Node selectorとは異なり、「可能ならスケジュールする」という選択が可能。
- `nodeAffinity`には、`requiredDuringSchedulingIgnoredDuringExecution`または`preferredDuringSchedulingIgnoredDuringExecution`を指定することができる。
- `requiredDuringSchedulingIgnoredDuringExecution`
    - 対応するNodeが見つからない場合、Podをスケジュールしない。(Node selectorと同じ挙動)
- `preferredDuringSchedulingIgnoredDuringExecution`
    - 対応するNodeが見つからない場合、適当なNodeにスケジュールする。

#### `Pod Affinity`と`Pod Anti-Affinity`
- Pod間のaffinity設定。
- 同じアプリケーションを動かしているPod同士を別のNodeにスケジュールする、といった設定が可能。

#### Podを分散するための設定: `Pod Topology Spread Constaints`
- Podを分散させるための設定。
- たとえば`topologyKey`にNodeの`kubernetes.io/hostname`ラベルを指定するとホスト間でPodを分散してスケジュールできる。
- `maxSkew`: Node間のPodの最大差分を指定することで分散の仕方を調整できる。

#### `Taint`と`Toleration`
- TaintとTolerationはそれぞれ対になる概念。
- TaintはNodeに付与する設定で、TolerationはPodに付与する設定。
- Taint/Tolerationは「あるNodeが特定のPodにしかスケジュールしたくない（とくに指定のないPodをスケジュールを拒否したい）」といった指定方法になる。（あまりわからなかったがいったんスキップ）

#### `Pod Priority`と`Preemption`
- PodにはPriority(優先度)を設定することができる。
- 優先度は`PriorityClass`というリソースを利用して指定する。
- `priorityClassName`を指定したPodがどのNodeにもスケジュールできないときにpreemption(追い出し)が発生する
    - あるNode上にスケジュールされているPodのうち、より優先度の低いPodをEvict(強制退去)させることで優先度の高いPodをスケジュール可能にする。
- Kubernetesでは`system-cluster-critical`と`system-node-critical`というPriorityClassがデフォルトで作成される。

```sh
$ kubectl describe pc
Name:              system-cluster-critical
Value:             2000000000
GlobalDefault:     false
PreemptionPolicy:  PreemptLowerPriority
Description:       Used for system critical pods that must run in the cluster, but can be moved to another node if necessary.
Annotations:       <none>
Events:            <none>


Name:              system-node-critical
Value:             2000001000
GlobalDefault:     false
PreemptionPolicy:  PreemptLowerPriority
Description:       Used for system critical pods that must not be moved from their current node.
Annotations:       <none>
Events:            <none>
```

20億...でかい。

### `Horizontal Pod Autoscaler`(HPA)
- Podを水平スケールさせるためのリソース。
- 通常CPUやメモリの値に応じてPodの数を増減させるが、任意のメトリクスを利用することも可能。
- HPAを利用するためにはmetrics-serverをインストールする必要がある。

負荷を掛けて水平スケールさせている様子。

<img width="4004" height="3284" alt="Image" src="https://github.com/user-attachments/assets/9f0786ee-20c1-4c9a-a935-c2a43941eb73" />

### `Vertical Pod Autoscaler`(VPA)
HPAとは同時に使用できない。(HPAのみ利用するケースが多い)

### アプリケーションの可用性を保証する`PodDisruptionBudget(PDB)`
- Nodeが故障したりKubernetesのバージョン更新などでNodeのシャットダウンが必要になることがある。
- DeploymentでカバーできるのはあくまでPodを更新するときのみ。
- NodeをメンテナンスするためにNodeからPodを安全に退避させるための機能の1つがPod Disruption Budget(PDB)。

以下のような設定値がある。
- `minAvailable`: 最低いくつのPodが利用可能な状態にするか
- `maxUnavailable`: 最低いくつのPodが利用不可能な状態になっていいか

Kubernetesはこれらの指定された値をみて、退避させるPodの数を制御してくれる。

## Chapter 9 Kubernetesの仕組み、アーキテクチャを理解しよう
![Image](https://github.com/user-attachments/assets/5af3798d-6c4c-48bb-b170-1d48cc2f75bc)

### Control Plane


- kube-apiserverはRESTで通信可能なAPIサーバ
- etcdは分散型KVS。
- -> コントロールプレーンはAPIサーバとデータベースでできている。
- kube-apiserverはユーザー(kubectl)からリクエストを受けてetcdにデータを保存したり、etcdに保存してあるデータを取得してユーザーに返したりしている。
- kube-schedulerはPodをNodeにスケジュールする役割を担っている。
- kube-contoller-managerはKubernetesを最低限動かすために必要な複数のコントローラを動かしている。

### Worker Node
- Worker Nodeは実際にアプリケーションコンテナの起動を行うNode。
- kubelet
    - クラスタ内の各Node上で動作している。
    - Podに紐づくコンテナを管理する。
    - kubeletが起動しているNodeにPodがスケジュールされるとコンテナランタイムに指示してコンテナを起動する。
- kube-proxy
    - Kubernetes Serviceリソースなどに応じてネットワーク設定を行うコンポーネント。
    - クラスタ内の各Node上で動作している。
    - kube-proxyによってクラスタ内外のネットワークセッションからPodへのネットワーク通信が可能になる。
- コンテナランタイム
    - コンテナを実行する役割のソフトウェア。

### kubectl
- kubectlはkube-apiserverと通信するためのCLIツール。
- kubectlはkube-apiserver間はJSONでやりとりをするが、kubectlはユーザーとyaml形式でやりとりできるように変換してくれている。

## Chapter 11 オブザーバビリティとモニタリングに触れてみよう
Prometheusで`go_gc_duration_seconds`のグラフを表示した様子。

<img width="2992" height="3158" alt="CleanShot 2025-12-08 at 01 54 05@2x" src="https://github.com/user-attachments/assets/3a7f9f09-1bf8-42d2-8d41-dac05b391db9" />

書籍の手順だとローカル起動したGrafanaにログインする際にパスワードが間違っていると表示された。
以下のコマンドでパスワードを取得してログインしたら入れた。

```sh
kubectl get secret --namespace monitoring kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```

Grafanaで`go_gc_duration_seconds`のグラフを表示した様子。

<img width="2992" height="3158" alt="CleanShot 2025-12-08 at 02 00 14@2x" src="https://github.com/user-attachments/assets/741d4ff0-670a-477e-a7e7-20c7360a97b6" />

