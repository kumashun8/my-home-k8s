## cluster の作成

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

```
root@koutei:~# kubeadm init --pod-network-cidr=10.10.0.0/16 --control-plane-endpoint=192.168.11.17
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

### DaemonSet calico-node が壊れる

放置してたら Mac から `kubectl` が打てなくなっていたので一から構築し直すと、calico-node という DamonSet が 1 台 not readly になってしまった。

```
❯ k get po -n calico-system -o wide
NAME                                       READY   STATUS    RESTARTS      AGE   IP              NODE      NOMINATED NODE   READINESS GATES
calico-kube-controllers-67f85d7449-chwvn   1/1     Running   1 (77m ago)   47h   10.10.111.196   adelie    <none>           <none>
calico-node-25lgz                          1/1     Running   1 (77m ago)   47h   10.0.0.12       iwatobi   <none>           <none>
calico-node-8dmrl                          0/1     Running   1 (20h ago)   47h   10.0.0.11       koutei    <none>           <none>
calico-node-fvpzm                          1/1     Running   1 (77m ago)   47h   10.0.0.13       adelie    <none>           <none>
calico-typha-649bcb5b9d-dg9wt              1/1     Running   2 (76m ago)   47h   10.0.0.13       adelie    <none>           <none>
calico-typha-649bcb5b9d-zjx65              1/1     Running   2 (76m ago)   47h   10.0.0.12       iwatobi   <none>           <none>

❯ k get ds -n calico-system
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
calico-node   3         3         2       3            2           kubernetes.io/os=linux   47h

```

ひとまずログを確認する。

```
❯ k logs calico-node-8dmrl -n calico-system
2022-03-14 11:14:02.496 [INFO][9] startup/startup.go 425: Early log level set to info
2022-03-14 11:14:02.505 [INFO][9] startup/utils.go 127: Using NODENAME environment for node name koutei
2022-03-14 11:14:02.508 [INFO][9] startup/utils.go 139: Determined node name: koutei
2022-03-14 11:14:02.508 [INFO][9] startup/startup.go 94: Starting node koutei with version v3.22.1
2022-03-14 11:14:02.595 [INFO][9] startup/startup.go 430: Checking datastore connection
2022-03-14 11:14:02.694 [INFO][9] startup/startup.go 454: Datastore connection verified
2022-03-14 11:14:02.694 [INFO][9] startup/startup.go 104: Datastore is ready
2022-03-14 11:14:02.853 [INFO][9] startup/autodetection_methods.go 103: Using autodetected IPv4 address on interface wlan0: 192.168.11.17/24
2022-03-14 11:14:02.853 [INFO][9] startup/startup.go 699: No AS number configured on node resource, using global value
2022-03-14 11:14:02.896 [INFO][9] startup/startup.go 748: found v6= in the kubeadm config map
2022-03-14 11:14:03.036 [INFO][9] startup/startup.go 674: FELIX_IPV6SUPPORT is false through environment variable
2022-03-14 11:14:03.164 [INFO][9] startup/startup.go 218: Using node name: koutei
2022-03-14 11:14:03.165 [INFO][9] startup/utils.go 191: Setting NetworkUnavailable to false
2022-03-14 11:14:03.839 [INFO][48] tunnel-ip-allocator/allocateip.go 304: Current address is still valid, do nothing currentAddr="10.10.140.129" type="vxlanTunnelAddress"
Calico node started successfully
bird: Unable to open configuration file /etc/calico/confd/config/bird.cfg: No such file or directory
bird: Unable to open configuration file /etc/calico/confd/config/bird6.cfg: No such file or directory
2022-03-14 11:14:05.394 [INFO][81] cni-config-monitor/token_watch.go 40: Watching contents for changes. directory="/var/run/secrets/kubernetes.io/serviceaccount/"
2022-03-14 11:14:05.508 [INFO][85] tunnel-ip-allocator/config.go 60: Found FELIX_TYPHAK8SSERVICENAME=calico-typha
2022-03-14 11:14:05.515 [INFO][85] tunnel-ip-allocator/config.go 60: Found FELIX_TYPHAK8SNAMESPACE=calico-system
2022-03-14 11:14:05.516 [INFO][85] tunnel-ip-allocator/config.go 60: Found FELIX_TYPHAKEYFILE=/felix-certs/key.key
2022-03-14 11:14:05.517 [INFO][85] tunnel-ip-allocator/config.go 60: Found FELIX_TYPHACERTFILE=/felix-certs/cert.crt
2022-03-14 11:14:05.517 [INFO][85] tunnel-ip-allocator/config.go 60: Found FELIX_TYPHACAFILE=/typha-ca/caBundle
2022-03-14 11:14:05.517 [INFO][85] tunnel-ip-allocator/config.go 60: Found FELIX_TYPHACN=typha-server
2022-03-14 11:14:05.582 [INFO][85] tunnel-ip-allocator/discovery.go 163: Found ready Typha addresses. addrs=[]string{"10.0.0.12:5473", "10.0.0.13:5473"}
2022-03-14 11:14:05.584 [INFO][85] tunnel-ip-allocator/discovery.go 166: Chose Typha to connect to. choice="10.0.0.13:5473"
2022-03-14 11:14:05.584 [INFO][85] tunnel-ip-allocator/startsyncerclient.go 56: Connecting to Typha. addr="10.0.0.13:5473"
2022-03-14 11:14:05.585 [INFO][85] tunnel-ip-allocator/sync_client.go 71:  requiringTLS=true
2022-03-14 11:14:05.585 [INFO][85] tunnel-ip-allocator/sync_client.go 200: Starting Typha client
2022-03-14 11:14:05.585 [INFO][85] tunnel-ip-allocator/sync_client.go 71:  requiringTLS=true
2022-03-14 11:14:05.587 [INFO][85] tunnel-ip-allocator/tlsutils.go 39: Make certificate verifier requiredCN="typha-server" requiredURISAN="" roots=&x509.CertPool{byName:map[string][]int{"0,1*0(\x06\x03U\x04\x03\f!tigera-operator-signer@1647091055":[]int{0}}, lazyCerts:[]x509.lazyCert{x509.lazyCert{rawSubject:[]uint8{0x30, 0x2c, 0x31, 0x2a, 0x30, 0x28, 0x6, 0x3, 0x55, 0x4, 0x3, 0xc, 0x21, 0x74, 0x69, 0x67, 0x65, 0x72, 0x61, 0x2d, 0x6f, 0x70, 0x65, 0x72, 0x61, 0x74, 0x6f, 0x72, 0x2d, 0x73, 0x69, 0x67, 0x6e, 0x65, 0x72, 0x40, 0x31, 0x36, 0x34, 0x37, 0x30, 0x39, 0x31, 0x30, 0x35, 0x35}, getCert:(func() (*x509.Certificate, error))(0x6e9900)}}, haveSum:map[x509.sum224]bool{x509.sum224{0xc, 0xf1, 0x62, 0x82, 0x2, 0xe2, 0x44, 0x29, 0x3f, 0x8, 0xc7, 0xa7, 0x6a, 0x86, 0x46, 0xb2, 0x2a, 0x95, 0xf1, 0x1c, 0xe0, 0x2c, 0x48, 0xf6, 0x50, 0x46, 0x36, 0xb6}:true}}
2022-03-14 11:14:05.588 [INFO][85] tunnel-ip-allocator/sync_client.go 252: Connecting to Typha. address="10.0.0.13:5473" connID=0x0 type="tunnel-ip-allocation"
2022-03-14 11:14:05.590 [FATAL][85] tunnel-ip-allocator/startsyncerclient.go 73: Failed to connect to Typha error=dial tcp 10.0.0.13:5473: connect: connection refused
2022-03-14 11:14:05.606 [INFO][87] status-reporter/startup.go 425: Early log level set to info
```

```
[root@adelie /]# /bin/calico-node -bird-ready -felix-ready
2022-03-14 13:20:37.835 [INFO][1383] confd/health.go 180: Number of node(s) with BGP peering established = 1

[root@iwatobi /]# /bin/calico-node -bird-ready -felix-ready
2022-03-14 13:31:13.452 [INFO][2427] confd/health.go 180: Number of node(s) with BGP peering established = 1

[root@koutei /]# /bin/calico-node -bird-ready -felix-ready
2022-03-14 13:31:56.463 [INFO][2785] confd/health.go 180: Number of node(s) with BGP peering established = 0
calico/node is not ready: BIRD is not ready: BGP not established with 192.168.11.19,192.168.11.18

```

```
❯ calicoctl get nodes --allow-version-mismatch
NAME
adelie
iwatobi
koutei

ubuntu@koutei:~$ sudo calicoctl node status
Calico process is running.

IPv4 BGP status
+---------------+-------------------+-------+----------+---------+
| PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |  INFO   |
+---------------+-------------------+-------+----------+---------+
| 192.168.11.19 | node-to-node mesh | start | 13:10:42 | Passive |
| 192.168.11.18 | node-to-node mesh | start | 13:10:42 | Passive |
+---------------+-------------------+-------+----------+---------+

IPv6 BGP status
No IPv6 peers found.

```

```
ubuntu@koutei:~$ calicoctl get node iwatobi -o yaml --allow-version-mismatch
apiVersion: projectcalico.org/v3
kind: Node
metadata:
  annotations:
    projectcalico.org/kube-labels: '{"beta.kubernetes.io/arch":"arm64","beta.kubernetes.io/os":"linux","kubernetes.io/arch":"arm64","kubernetes.io/hostname":"iwatobi","kubernetes.io/os":"linux"}'
  creationTimestamp: "2022-03-12T13:07:52Z"
  labels:
    beta.kubernetes.io/arch: arm64
    beta.kubernetes.io/os: linux
    kubernetes.io/arch: arm64
    kubernetes.io/hostname: iwatobi
    kubernetes.io/os: linux
  name: iwatobi
  resourceVersion: "258169"
  uid: a446b740-8a8e-4550-b669-bf13c130d76b
spec:
  addresses:
  - address: 192.168.11.18/24
    type: CalicoNodeIP
  - address: 10.0.0.12
    type: InternalIP
  bgp:
    ipv4Address: 192.168.11.18/24
  ipv4VXLANTunnelAddr: 10.10.183.65
  orchRefs:
  - nodeName: iwatobi
    orchestrator: k8s
status:
  podCIDRs:
  - 10.10.1.0/24
ubuntu@koutei:~$ calicoctl get node adelie -o yaml --allow-version-mismatch
apiVersion: projectcalico.org/v3
kind: Node
metadata:
  annotations:
    projectcalico.org/kube-labels: '{"beta.kubernetes.io/arch":"arm64","beta.kubernetes.io/os":"linux","kubernetes.io/arch":"arm64","kubernetes.io/hostname":"adelie","kubernetes.io/os":"linux"}'
  creationTimestamp: "2022-03-12T13:07:58Z"
  labels:
    beta.kubernetes.io/arch: arm64
    beta.kubernetes.io/os: linux
    kubernetes.io/arch: arm64
    kubernetes.io/hostname: adelie
    kubernetes.io/os: linux
  name: adelie
  resourceVersion: "258194"
  uid: 9d70c247-c150-45ac-a8d9-d5b85ffbb3ea
spec:
  addresses:
  - address: 192.168.11.19/24
    type: CalicoNodeIP
  - address: 10.0.0.13
    type: InternalIP
  bgp:
    ipv4Address: 192.168.11.19/24
  ipv4VXLANTunnelAddr: 10.10.111.192
  orchRefs:
  - nodeName: adelie
    orchestrator: k8s
status:
  podCIDRs:
  - 10.10.2.0/24
ubuntu@koutei:~$ calicoctl get node koutei -o yaml --allow-version-mismatch
apiVersion: projectcalico.org/v3
kind: Node
metadata:
  annotations:
    projectcalico.org/kube-labels: '{"beta.kubernetes.io/arch":"arm64","beta.kubernetes.io/os":"linux","kubernetes.io/arch":"arm64","kubernetes.io/hostname":"koutei","kubernetes.io/os":"linux","node-role.kubernetes.io/control-plane":"","node-role.kubernetes.io/master":"","node.kubernetes.io/exclude-from-external-load-balancers":""}'
  creationTimestamp: "2022-03-12T13:05:30Z"
  labels:
    beta.kubernetes.io/arch: arm64
    beta.kubernetes.io/os: linux
    kubernetes.io/arch: arm64
    kubernetes.io/hostname: koutei
    kubernetes.io/os: linux
    node-role.kubernetes.io/control-plane: ""
    node-role.kubernetes.io/master: ""
    node.kubernetes.io/exclude-from-external-load-balancers: ""
  name: koutei
  resourceVersion: "258165"
  uid: 9a77d9ea-908b-4d34-9ddb-13167893b8f6
spec:
  addresses:
  - address: 192.168.11.17/24
    type: CalicoNodeIP
  - address: 10.0.0.11
    type: InternalIP
  bgp:
    ipv4Address: 192.168.11.17/24
  ipv4VXLANTunnelAddr: 10.10.140.128
  orchRefs:
  - nodeName: koutei
    orchestrator: k8s
status:
  podCIDRs:
  - 10.10.0.0/24
```
