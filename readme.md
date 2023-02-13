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
    ```sh
    # metrics serverをデプロイする
    ```
    [./metrics_server/component.yaml](./metrics_server/components.yaml)をapplyして、

1. deploymentで3つのpodをdeployする。pod内容はcpuをガンガンに上げるもの。
    ``` sh
    # deploy
    $ kubectl apply -f ./hpa_sample/deployment.yaml 

    # get pod
    $ kubectl get pods
    NAME                              READY   STATUS    RESTARTS   AGE
    hpa-test-deploy-9c464d947-2gtpp   1/1     Running   0          23s
    hpa-test-deploy-9c464d947-fjwhv   1/1     Running   0          23s
    hpa-test-deploy-9c464d947-wl7x9   1/1     Running   0          23s

    # kubectl topでCPU負荷を確認する
    NAME                               CPU(cores)   MEMORY(bytes)   
    hpa-test-deploy-7649c8cffb-22zc7   1000m        0Mi
    hpa-test-deploy-7649c8cffb-8jt2r   1001m        0Mi
    hpa-test-deploy-7649c8cffb-w2kwj   1001m        0Mi    
    ```

2. 水平ポッド自動スケーラをデプロイする。
    ```sh
    # スケーラをデプロイする
    $ kubectl apply -f ./hpa_sample/hpa.yaml
    # スケール状況を確認する
    $ kubectl get HorizontalPodAutoscaler hpa-test
    ```

    NAME       REFERENCE                    TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
    hpa-test   Deployment/hpa-test-deploy   <unknown>/50%   1         5         3          79s

### 参考サイト
- Qiita [KubernetesのPodとNodeのAuto Scalingについて](https://qiita.com/sheepland/items/37ea0b77df9a4b4c9d80)
- Zenn [kubectl topを使えるようにする](https://zenn.dev/hkw/articles/0ee0f726008a63)
- 公式 [水平ポッド自動スケーラ(v2) リファレンス](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/horizontal-pod-autoscaler-v2/)