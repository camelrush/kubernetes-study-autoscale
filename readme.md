# Kubernetes AutoScale

Kubernetesを使用したポッドのオートスケーリングを実践。

## スケール対象
Kubernetesのリソースのうち、以下の「scaled resource object」を対象とする。

  - Deployment
  - ReplicaSet
  - ReplicationController
  - StatefulSet

<br>

## 垂直/水平ポッドスケール
- 垂直ポッドスケールは、ノード内でポッドの使用スペック(CPU/メモリ)を拡縮すること。
- 水平ポッドスケールは、ノード内でポッドの数を増減すること。
- 本稿では、水平スケールの実施を行う。垂直スケール(VirticalPodAutoScaler)については、[こちら](https://qiita.com/shmurata/items/197b5b722ac7e9dedb90)を参照のこと。

<br>

## スケール指定 hpa（HorizontalPodAutoscaler）

- 各スケールリソースに対して、HorizontalPodAutoscalerオブジェクトを作成する。

  - 書式（hpa.yaml）

    ``` yaml
    apiVersion: autoscaling/v2beta1
    kind: HorizontalPodAutoscaler
    metadata:
    name: php-hpa
    namespace: default
    spec:
    scaleTargetRef: # ここでautoscale対象となる`scaled resource object`を指定
        apiVersion: apps/v1
        kind: Deployment
        name: php-deploy
    minReplicas: 1 # 最小レプリカ数
    maxReplicas: 5 # 最大レプリカ数
    metrics:
    - type: Resource
        resource:
        name: cpu
        targetAverageUtilization: 50 # CPU使用率が常に50%になるように指定    
    ```

## 実践

### 注意事項

0. 事前準備  
    本稿では、CPU負荷率を確認するため、`kubectl top`コマンドを使用する。   
    これを使用するために、環境のk8sにmetrics-serverをインストールする必要がある。  
    詳細は[こちらのgithub](https://github.com/kubernetes-sigs/metrics-server/)を参照
    ```sh
    # metrics serverをデプロイする
    $ kubectl apply -f ./metrics_server/component.yaml
    ```

1. deploymentで3つのpodをdeployする。
    ``` sh
    # デプロイする(初期レプリカは3)
    $ kubectl apply -f ./hpa_sample/deployment.yaml 

    # Podが3つデプロイされている
    # pod内容はcpuをガンガンに上げるもの。
    $ kubectl get pods
    NAME                               READY   STATUS    RESTARTS   AGE
    hpa-test-deploy-7649c8cffb-2b2xm   1/1     Running   0          11s
    hpa-test-deploy-7649c8cffb-c4dzh   1/1     Running   0          11s
    hpa-test-deploy-7649c8cffb-qc22b   1/1     Running   0          11s

    # top podを実行すると、各podがCPUを大きく使っていることがわかる。
    $ kubectl top pod
    NAME                              CPU(cores)   MEMORY(bytes)   
    hpa-test-deploy-7649c8cffb-2b2xm   996m         0Mi             
    hpa-test-deploy-7649c8cffb-c4dzh   995m         0Mi             
    hpa-test-deploy-7649c8cffb-qc22b   995m         0Mi           

    # top nodeでnode全体のCPU負荷を確認する（起動初期は低い）
    $ kubectl top node
    NAME             CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
    docker-desktop   204m         5%     1553Mi          82%       

    # 少し時間を置いて再実行するとCPU負荷率が上がっている。
    $ kubectl top node
    NAME             CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
    docker-desktop   2683m        67%    1513Mi          80%            
    ```

2. 水平ポッド自動スケーラをデプロイする。
    ```sh
    # スケーラをデプロイする
    $ kubectl apply -f ./hpa_sample/hpa.yaml

    # 1分ほど待つ

    # podが5つ(Max)までスケールアウトしていることを確認
    $ kubectl get pod
    NAME                              READY   STATUS    RESTARTS   AGE
    hpa-test-deploy-d89d9c8d9-dbkq9   1/1     Running   0          3m48s
    hpa-test-deploy-d89d9c8d9-fphvh   1/1     Running   0          36s
    hpa-test-deploy-d89d9c8d9-pgn9f   1/1     Running   0          3m48s
    hpa-test-deploy-d89d9c8d9-ss8dp   1/1     Running   0          36s
    hpa-test-deploy-d89d9c8d9-tl645   1/1     Running   0          3m48s

    # スケール状況を確認する(5つまでスケールしている)
    $ kubectl get HorizontalPodAutoscaler hpa-test
    NAME       REFERENCE                    TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
    hpa-test   Deployment/hpa-test-deploy   390%/50%   1         5         5          4m52s

    ```

### 参考サイト
- Qiita [KubernetesのPodとNodeのAuto Scalingについて](https://qiita.com/sheepland/items/37ea0b77df9a4b4c9d80)
- Zenn [kubectl topを使えるようにする](https://zenn.dev/hkw/articles/0ee0f726008a63)
- 公式 [水平ポッド自動スケーラ(v2) リファレンス](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/horizontal-pod-autoscaler-v2/)