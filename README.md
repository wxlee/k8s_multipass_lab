# 使用Multipass與Cloud-init快速創建Kubernetes Lab環境

Env: 
- macOS 10.15.7 (僅在Intel CPU測試過)
- VM: Ubuntu 22.04
- K8s: 1.26.3
- Calico: 3.25.1
- Docker: 23.0.2
- cri-dockerd: 0.3.1


```bash
# [mac] macOS安裝multipass
brew install multipass

# [mac] 下載cloud-init yaml
wget https://raw.githubusercontent.com/wxlee/k8s_multipass_lab/main/master-cloud-init.yaml

# [mac] 使用cloud-init進行基本設置，使用docker, cri-dockerd, containerd
multipass launch 22.04 -n k8s-master --cloud-init master-cloud-init.yaml -c 2 -m 4G --disk 20G

# [mac] multipass執行過程，可以透過shell環境去觀察cloud-init執行狀況
#   登入VM k8s-master
multipass shell k8s-master

# [ubuntu] 切換root角色進行後續動作
sudo su - root

# [ubuntu] k8s-master中，查看cloud-init日誌
tail -f /var/log/cloud-init-output.log

# 出現類似下述內容表示cloud init完成
Cloud-init v. 22.4.2-0ubuntu0~22.04.1 finished at Mon, 03 Apr 2023 10:31:41 +0000. Datasource DataSourceNoCloud [seed=/dev/sr0][dsmode=net].  Up 240.79 seconds

# [ubuntu] 安裝完畢進行初始化
kubeadm init --pod-network-cidr=10.1.0.0/16 --cri-socket unix:///var/run/cri-dockerd.sock

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# [ubuntu] 安裝calico
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

# [ubuntu] 查看calico pod初始化狀態，完成後STATUS都會出現Running
watch kubectl get pods -n calico-system

# [ubuntu] 查看node狀態，應為Ready
kubectl get node

NAME         STATUS   ROLES           AGE   VERSION
k8s-master   Ready    control-plane   13m   v1.26.3

# [ubuntu] 移除master污點設置，讓pod可以派發到這個節點
kubectl taint nodes k8s-master node-role.kubernetes.io/control-plane:NoSchedule-

# [ubuntu] 創建一個deployment nginx-app
kubectl create deployment nginx-app --image=nginx

deployment.apps/nginx-app created

kubectl get pod,svc -o wide

NAME                            READY   STATUS    RESTARTS   AGE   IP             NODE         NOMINATED NODE   READINESS GATES
pod/nginx-app-5d47bf8b9-fp9lc   1/1     Running   0          90s   10.1.235.199   k8s-master   <none>           <none>

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   16m   <none>

# [ubuntu] 測試訪問該pod，可正常訪問nginx
curl -I 10.1.235.199

HTTP/1.1 200 OK
Server: nginx/1.23.4
Date: Mon, 03 Apr 2023 10:52:00 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 28 Mar 2023 15:01:54 GMT
Connection: keep-alive
ETag: "64230162-267"
Accept-Ranges: bytes

# [mac] 清除VM
multipass delete k8s-master
multipass purge
```

# 新增k8s指令補全與色彩
```bash
kubectl completion bash > ~/.kube/k8s_bash_completion.sh
echo -e "\n#kubectl shell completion\nsource '$HOME/.kube/k8s_bash_completion.sh'\n" >> $HOME/.bash_profile
echo -e "source '/root/.bashrc'" >> $HOME/.bash_profile
source $HOME/.bash_profile
wget https://github.com/hidetatz/kubecolor/releases/download/v0.0.25/kubecolor_0.0.25_Linux_x86_64.tar.gz
tar zxf kubecolor_0.0.25_Linux_x86_64.tar.gz
echo "alias kubectl='/root/kubecolor'" >> ~/.bashrc
. ~/.bashrc
complete -o default -F __start_kubectl kubecolor
```


# 參考資料

[Multipass + Kubernetes + Calico](https://medium.com/@ypelud/multipass-kubernetes-calico-30366d293162)

[Calico](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart#operator-based-installation)

[ngaffa/cloud-init.yaml](https://gist.github.com/ngaffa/15d46c98dd82620c8120ddf7398d6dbd)

[Updated: Dockershim Removal FAQ](https://kubernetes.io/blog/2022/02/17/dockershim-faq/)
