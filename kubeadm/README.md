## Kubeadm セットアップの道

ref : https://kubernetes.io/ja/docs/setup/production-environment/tools/kubeadm/_print/

### 事前確認

```
ubuntu@koutei:~$ sudo modprobe br_netfilter
ubuntu@koutei:~$ lsmod | grep br_netfilter
br_netfilter           32768  0
bridge                278528  1 br_netfilter

ubuntu@koutei:~$ sudo apt-get install -y iptables arptables ebtables
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
ebtables is already the newest version (2.0.11-4).
iptables is already the newest version (1.8.7-1ubuntu2).
arptables is already the newest version (0.0.5-3).
0 upgraded, 0 newly installed, 0 to remove and 1 not upgraded.
ubuntu@koutei:~$ sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
ubuntu@koutei:~$ sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
ubuntu@koutei:~$ sudo update-alternatives --set arptables /usr/sbin/arptables-legacy
ubuntu@koutei:~$ sudo update-alternatives --set ebtables /usr/sbin/ebtables-legacy
ubuntu@koutei:~$
```

### Docker のインストール

https://kubernetes.io/ja/docs/setup/production-environment/container-runtimes/

doc と違ったところ。

```
ubuntu@koutei:~$ add-apt-repository \
  "deb [arch=arm64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"

ubuntu@koutei:~$ sudo apt-get update && \
  sudo apt-get install -y docker-ce docker-ce-cli
```

結果

```
ubuntu@koutei:~$ sudo docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

```

それ以降はすんなりインストールできた。
