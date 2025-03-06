## Day1：kindを使ってローカル環境でKubernetesのクラスタを構築する

### 仕組み

ローカル環境にDockerコンテナを起動し、それをノードに見立てる。

### 手順

`cluster.yaml` を作成。

```
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
nodes:
- role: control-plane
  image: kindest/node:v1.27.2
  extraPortMappings:
  - containerPort: 30080
    hostPort: 80
- role: worker
  image: kindest/node:v1.27.2
- role: worker
  image: kindest/node:v1.27.2
- role: worker
  image: kindest/node:v1.27.2
```

その後、以下のコマンドを実行することでローカル環境にKubernetesクラスタが構築される。

```
$ kind create cluster --config cluster.yaml --name kindcluster
Creating cluster "kindcluster" ...
 ✓ Ensuring node image (kindest/node:v1.27.2) 🖼
 ✓ Preparing nodes 📦 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-kindcluster"
You can now use your cluster with:

kubectl cluster-info --context kind-kindcluster

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
```

```
$ kubectl get nodes
NAME                        STATUS   ROLES           AGE   VERSION
kindcluster-control-plane   Ready    control-plane   50s   v1.27.2
kindcluster-worker          Ready    <none>          32s   v1.27.2
kindcluster-worker2         Ready    <none>          32s   v1.27.2
kindcluster-worker3         Ready    <none>          31s   v1.27.2
```

```
$ docker container ls
CONTAINER ID   IMAGE                  COMMAND                   CREATED         STATUS         PORTS                                              NAMES
d488b6b8610b   kindest/node:v1.27.2   "/usr/local/bin/entr…"   2 minutes ago   Up 2 minutes                                                      kindcluster-worker2
754aa8ff90a3   kindest/node:v1.27.2   "/usr/local/bin/entr…"   2 minutes ago   Up 2 minutes                                                      kindcluster-worker3
ba2f7773e3a5   kindest/node:v1.27.2   "/usr/local/bin/entr…"   2 minutes ago   Up 2 minutes                                                      kindcluster-worker
b8fb759511c0   kindest/node:v1.27.2   "/usr/local/bin/entr…"   2 minutes ago   Up 2 minutes   127.0.0.1:54454->6443/tcp, 0.0.0.0:80->30080/tcp   kindcluster-control-plane
```

### 参考情報

- https://www.ios-net.co.jp/blog/20230607-1128/
- https://kind.sigs.k8s.io/
