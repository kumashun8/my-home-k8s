# my-home-k8s

おうち k8s の構築メモ。  
基本的にはこちらの手順に沿って環境構築を行なっています。  
https://github.com/CyberAgentHack/home-kubernetes-2020

## 物理構成

以下で紹介されているパーツを使って構築しました。  
https://github.com/CyberAgentHack/home-kubernetes-2020/tree/master/how-to-create-cluster-physical

<img src="https://raw.githubusercontent.com/kumashun8/my-home-k8s/main/images/raspi-cluster.jpeg" width="400" alt="raspi-cluster">

## 論理構成

物理構成に続いて同 repogitory の内容を参考にしました。  
https://github.com/CyberAgentHack/home-kubernetes-2020/tree/master/how-to-create-cluster-logical-hardway

### ネットワーク

- Node 用サブネット
  - Ethernet
  - 10.0.0.0/24
    - Node1: 10.0.0.11
    - Node2: 10.0.0.12
    - Node3: 10.0.0.13
- Node 外部通信用サブネット (例)
  - 無線 LAN
  - 192.168.11.0/24
    - Node1: 192.168.11.17
    - Node2: 192.168.11.18
    - Node3: 192.168.11.19

### ホスト名

なんとなくペンギン縛り。

- Node1: **koutei**
- Node2: **iwatobi**
- Node3: **adelie**

### バックアップ

```
ubuntu@iwatobi:~$ sudo fdisk -l
[snip]

Disk /dev/mmcblk0: 29.72 GiB, 31914983424 bytes, 62333952 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xb59eb0cd

ubuntu@iwatobi:~$ sudo dd if=/dev/mmcblk0 of=raspi.img bs=10M
(実行中...)
```
