# 使用Multipass與Cloud-init快速創建Kubernetes Lab環境

Env: macOS 10.15.7 (Intel CPU)

```bash
# macOS安裝multipass
brew install multipass

# 使用cloud-init進行基本設置，使用docker, cri-dockerd, containerd
multipass launch 22.04 -n k8s-master --cloud-init master-cloud-init.yaml -c 2 -m 4G --disk 20G

# 安裝完畢進行初始化
kubeadm init --pod-network-cidr=10.1.0.0/16 --cri-socket unix:///var/run/cri-dockerd.sock

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 安裝calico
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/tigera-operator.yaml

kubectl create -f- <<EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 10.1.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
EOF

# 查看calico pod初始化狀態，完成後STATUS都會出現Running
watch kubectl get pods -n calico-system

# 查看node狀態，應為Ready
kubectl get node

NAME         STATUS   ROLES           AGE   VERSION
k8s-master   Ready    control-plane   13m   v1.26.3


# 移除master污點設置，讓pod可以派發到這個節點
kubectl taint nodes k8s-master node-role.kubernetes.io/control-plane:NoSchedule-

# 創建一個deployment nginx-app
kubectl create deployment nginx-app --image=nginx
deployment.apps/nginx-app created

root@k8s-master:~# kubectl get pod,svc -o wide
NAME                            READY   STATUS    RESTARTS   AGE   IP             NODE         NOMINATED NODE   READINESS GATES
pod/nginx-app-5d47bf8b9-fp9lc   1/1     Running   0          90s   10.1.235.199   k8s-master   <none>           <none>

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   16m   <none>

# 測試訪問該pod，可正常訪問nginx
root@k8s-master:~# curl -I 10.1.235.199
HTTP/1.1 200 OK
Server: nginx/1.23.4
Date: Mon, 03 Apr 2023 10:52:00 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 28 Mar 2023 15:01:54 GMT
Connection: keep-alive
ETag: "64230162-267"
Accept-Ranges: bytes

# 清除VM
multipass delete k8s-master
multipass purge
```

# 參考資料

[Multipass + Kubernetes + Calico](https://medium.com/@ypelud/multipass-kubernetes-calico-30366d293162)

[Calico](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart#operator-based-installation)

[ngaffa/cloud-init.yaml](https://gist.github.com/ngaffa/15d46c98dd82620c8120ddf7398d6dbd)
