## cluster の作成

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

```
root@koutei:~# kubeadm init --pod-network-cidr=10.10.0.0/16
(snip)

kubeadm join 192.168.11.17:6443 --token tj6aa3.ijpxk45pp2t2ba3b \
	--discovery-token-ca-cert-hash sha256:41bf4b71b614abc89f245269abd7ef1fc37821488a99f42cd49bdc3d6ada9f70

```

### Calico のインストール

https://docs.projectcalico.org/getting-started/kubernetes/quickstart

CIDR の指定を自分の設定と揃えて core-dns を再起動。

```
ubuntu@koutei:~$ k edit installation -n kube-system

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
      cidr: 192.168.0.0/16 => 10.10.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()

ubuntu@koutei:~$ k rollout restart deploy/core-dns -n kube-system

```

また Node の internal ip の割り当てが想定と違ったので、特定の Node にだけ以下の操作を行なった。

https://zenn.dev/liberaldays/articles/kube-multiple-nic-19c1fd1#1.%E3%83%8E%E3%83%BC%E3%83%89internal-ip%E3%81%AE%E5%A4%89%E6%9B%B4

```
root@iwatobi:~# sudo vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# 変更前
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# 変更後
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml --node-ip=10.0.0.12"

```

ひとまず core-dns が Running になったので、良しとする。

```
ubuntu@koutei:~$ k get po -n kube-system
NAME                             READY   STATUS    RESTARTS   AGE
coredns-576b945fcb-44zm7         1/1     Running   0          116s
coredns-576b945fcb-9tjsk         1/1     Running   0          115s
etcd-koutei                      1/1     Running   0          131m
kube-apiserver-koutei            1/1     Running   0          131m
kube-controller-manager-koutei   1/1     Running   0          131m
kube-proxy-27xrl                 1/1     Running   0          131m
kube-proxy-bgrdx                 1/1     Running   0          80m
kube-proxy-d6wnn                 1/1     Running   0          101m
kube-scheduler-koutei            1/1     Running   0          131m
```

ついでに cluster name も変えた。

```
ubuntu@koutei:~$ k config current-context
kubernetes-admin@polar-bear

```
