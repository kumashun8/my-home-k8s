## calico リトライ

https://projectcalico.docs.tigera.io/getting-started/kubernetes/hardway/overview

### 1. Stand up Kubernetes

clsuter の構築自体は終わっている。calico 関連のリソースは一掃して仕切り直し。

```
❯ k get po -A
NAMESPACE     NAME                             READY   STATUS    RESTARTS       AGE
kube-system   coredns-7bfcf44787-9mqf7         1/1     Running   0              49m
kube-system   coredns-7bfcf44787-cbj78         1/1     Running   0              49m
kube-system   etcd-koutei                      1/1     Running   3 (21h ago)    2d
kube-system   kube-apiserver-koutei            1/1     Running   3 (21h ago)    2d
kube-system   kube-controller-manager-koutei   1/1     Running   3 (21h ago)    2d
kube-system   kube-proxy-4l4hl                 1/1     Running   1 (166m ago)   2d
kube-system   kube-proxy-gx2pw                 1/1     Running   1 (166m ago)   2d
kube-system   kube-proxy-k8vh6                 1/1     Running   1 (21h ago)    2d
kube-system   kube-scheduler-koutei            1/1     Running   3 (21h ago)    2d

```

### 2. The Calico datastore

```
❯ kubectl apply -f https://projectcalico.docs.tigera.io/manifests/crds.yaml
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created

❯ calicoctl get nodes

NAME
adelie
iwatobi
koutei

❯ calicoctl get ippools

NAME   CIDR   SELECTOR


```

### 3. Configure IP pools

```
cat > pool1.yaml <<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: pool1
spec:
  cidr: 10.10.0.0/18
  ipipMode: Never
  natOutgoing: true
  disabled: false
  nodeSelector: all()
EOF

cat > pool2.yaml <<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: pool2
spec:
  cidr: 10.10.192.0/19
  ipipMode: Never
  natOutgoing: true
  disabled: true
  nodeSelector: all()
EOF

❯ calicoctl create -f pool1.yaml
Successfully created 1 'IPPool' resource(s)

❯ calicoctl create -f pool2.yaml
Successfully created 1 'IPPool' resource(s)

❯ calicoctl get ippools

NAME    CIDR             SELECTOR
pool1   10.10.0.0/18     all()
pool2   10.10.192.0/19   all()

```

### 4. Install CNI plugin

```
ubuntu@koutei:~$ openssl req -newkey rsa:4096 \
           -keyout cni.key \
           -nodes \
           -out cni.csr \
           -subj "/CN=calico-cni"
Generating a RSA private key
.................................................................................++++
..........................++++
writing new private key to 'cni.key'
-----

ubuntu@koutei:~$ sudo openssl x509 -req -in cni.csr \
                  -CA /etc/kubernetes/pki/ca.crt \
                  -CAkey /etc/kubernetes/pki/ca.key \
                  -CAcreateserial \
                  -out cni.crt \
                  -days 365
Signature ok
subject=CN = calico-cni
Getting CA Private Key

ubuntu@koutei:~$ kubectl config view -o jsonpath='{.clusters[0].cluster.server}'
https://192.168.11.17:6443

ubuntu@koutei:~$ kubectl config set-cluster kubernetes \
    --certificate-authority=/etc/kubernetes/pki/ca.crt \
    --embed-certs=true \
    --server=$APISERVER \
    --kubeconfig=cni.kubeconfig
Cluster "kubernetes" set.

ubuntu@koutei:~$ kubectl config set-credentials calico-cni \
    --client-certificate=cni.crt \
    --client-key=cni.key \
    --embed-certs=true \
    --kubeconfig=cni.kubeconfig
User "calico-cni" set.

ubuntu@koutei:~$ kubectl config set-context default \
    --cluster=kubernetes \
    --user=calico-cni \
    --kubeconfig=cni.kubeconfig
Context "default" created.

ubuntu@koutei:~$ kubectl config use-context default --kubeconfig=cni.kubeconfig
Switched to context "default".

ubuntu@koutei:~$ sudo scp cni.kubeconfig ubuntu@iwatobi:~/
ubuntu@iwatobi's password:
cni.kubeconfig                                                                              100% 7973     3.7MB/s   00:00

ubuntu@koutei:~$ sudo scp cni.kubeconfig ubuntu@adelie:~/
The authenticity of host 'adelie (10.0.0.13)' can't be established.
ECDSA key fingerprint is SHA256:PB2UJdihl0UvypsLvrO57ExJW+wWdqQDrvjhkVa08nU.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'adelie,10.0.0.13' (ECDSA) to the list of known hosts.
ubuntu@adelie's password:
cni.kubeconfig

ubuntu@koutei:~$ kubectl apply -f - <<EOF
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: calico-cni
rules:
  # The CNI plugin needs to get pods, nodes, and namespaces.
  - apiGroups: [""]
    resourcesś
      - pods
      - nodes
      - namespaces
    verbs:
      - get
  # The CNI plugin patches pods/status.
  - apiGroups: [""]
    resources:
      - pods/status
    verbs:
      - patch
 # These permissions are required for Calico CNI to perform IPAM allocations.
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - blockaffinities
      - ipamblocks
      - ipamhandles
    verbs:
      - get
      - list
      - create
      - update
      - delete
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - ipamconfigs
      - clusterinformations
      - ippools
    verbs:
      - get
      - list
EOF

clusterrole.rbac.authorization.k8s.io/calico-cni created

ubuntu@koutei:~$ kubectl create clusterrolebinding calico-cni --clusterrole=calico-cni --user=calico-cni
clusterrolebinding.rbac.authorization.k8s.io/calico-cni created

ubuntu@koutei:~$ sudo su
root@koutei:/home/ubuntu# curl -L -o /opt/cni/bin/calico https://github.com/projectcalico/cni-plugin/releases/download/v3.14.0/calico-amd64
chmod 755 /opt/cni/bin/calico
curl -L -o /opt/cni/bin/calico-ipam https://github.com/projectcalico/cni-plugin/releases/download/v3.14.0/calico-ipam-amd64
chmod 755 /opt/cni/bin/calico-ipam
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   649    0   649    0     0   1978      0 --:--:-- --:--:-- --:--:--  1978
100 37.7M  100 37.7M    0     0  5388k      0  0:00:07  0:00:07 --:--:-- 6319k
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   654  100   654    0     0   2006      0 --:--:-- --:--:-- --:--:--  2006
100 36.8M  100 36.8M    0     0  5102k      0  0:00:07  0:00:07 --:--:-- 5940k

root@koutei:/home/ubuntu# mkdir -p /etc/cni/net.d/
root@koutei:/home/ubuntu# ls
calicoctl.1  cni.crt  cni.csr  cni.key  cni.kubeconfig  snap
root@koutei:/home/ubuntu# cp cni.kubeconfig /etc/cni/net.d/calico-kubeconfig
chmod 600 /etc/cni/net.d/calico-kubeconfig
root@koutei:/home/ubuntu# cat > /etc/cni/net.d/10-calico.conflist <<EOF
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "kubernetes",
      "mtu": 1500,
      "ipam": {
          "type": "calico-ipam"
      },
      "policy": {
          "type": "k8s"
      },
      "kubernetes": {
          "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {
      "type": "portmap",
      "snat": true,
      "capabilities": {"portMappings": true}
    }
  ]
}
EOF
root@koutei:/home/ubuntu# exit
exit
ubuntu@koutei:~$ kubectl get nodes
NAME      STATUS   ROLES                  AGE    VERSION
adelie    Ready    <none>                 2d1h   v1.23.4
iwatobi   Ready    <none>                 2d1h   v1.23.4
koutei    Ready    control-plane,master   2d1h   v1.23.4
```

### 5. Install Typha

```
ubuntu@koutei:~$ openssl req -x509 -newkey rsa:4096 \
                  -keyout typhaca.key \
                  -nodes \
                  -out typhaca.crt \
                  -subj "/CN=Calico Typha CA" \
                  -days 365
Generating a RSA private key
.........................................++++
.............................................................................++++
writing new private key to 'typhaca.key'
-----

ubuntu@koutei:~$ kubectl create configmap -n kube-system calico-typha-ca --from-file=typhaca.crt

configmap/calico-typha-ca created

ubuntu@koutei:~$ openssl req -newkey rsa:4096 \
           -keyout typha.key \
           -nodes \
           -out typha.csr \
           -subj "/CN=calico-typha"
Generating a RSA private key
.............................++++
...........++++
writing new private key to 'typha.key'
-----

ubuntu@koutei:~$ openssl x509 -req -in typha.csr \
                  -CA typhaca.crt \
                  -CAkey typhaca.key \
                  -CAcreateserial \
                  -out typha.crt \
                  -days 365
Signature ok
subject=CN = calico-typha
Getting CA Private Key

ubuntu@koutei:~$ kubectl create secret generic -n kube-system calico-typha-certs --from-file=typha.key --from-file=typha.crt
secret/calico-typha-certs created

ubuntu@koutei:~$ kubectl create serviceaccount -n kube-system calico-typha
serviceaccount/calico-typha created
ubuntu@koutei:~$ kubectl apply -f - <<EOF
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: calico-typha
rules:
  - apiGroups: [""]
    resources:
      - pods
      - namespaces
      - serviceaccounts
      - endpoints
      - services
      - nodes
    verbs:
      # Used to discover service IPs for advertisement.
      - watch
      - list
  - apiGroups: ["networking.k8s.io"]
    resources:
      - networkpolicies
    verbs:
      - watch
      - list
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      - globalfelixconfigs
      - felixconfigurations
      - bgppeers
      - globalbgpconfigs
      - bgpconfigurations
      - ippools
      - ipamblocks
      - globalnetworkpolicies
      - globalnetworksets
      - networkpolicies
      - clusterinformations
      - hostendpoints
      - blockaffinities
      - networksets
    verbs:
      - get
      - list
      - watch
  - apiGroups: ["crd.projectcalico.org"]
    resources:
      #- ippools
      #- felixconfigurations
      - clusterinformations
    verbs:
      - get
      - create
      - update
EOF
clusterrole.rbac.authorization.k8s.io/calico-typha created

ubuntu@koutei:~$ kubectl create clusterrolebinding calico-typha --clusterrole=calico-typha --serviceaccount=kube-system:calico-typha
clusterrolebinding.rbac.authorization.k8s.io/calico-typha created

ubuntu@koutei:~$ kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: calico-typha
  namespace: kube-system
  labels:
    k8s-app: calico-typha
spec:
  replicas: 3
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      k8s-app: calico-typha
  template:
    metadata:
      labels:
        k8s-app: calico-typha
      annotations:
        cluster-autoscaler.kubernetes.io/safe-to-evict: 'true'
    spec:
      hostNetwork: true
      tolerations:
        # Mark the pod as a critical add-on for rescheduling.
        - key: CriticalAddonsOnly
          operator: Exists
      serviceAccountName: calico-typha
      priorityClassName: system-cluster-critical
      containers:
      - image: calico/typha:v3.8.0
        name: calico-typha
        ports:
        - containerPort: 5473
          name: calico-typha
          protocol: TCP
        env:
          # Disable logging to file and syslog since those don't make sense in Kubernetes.
          - name: TYPHA_LOGFILEPATH
            value: "none"
          - name: TYPHA_LOGSEVERITYSYS
            value: "none"
          # Monitor the Kubernetes API to find the number of running instances and rebalance
          # connections.
          - name: TYPHA_CONNECTIONREBALANCINGMODE
            value: "kubernetes"
          - name: TYPHA_DATASTORETYPE
            value: "kubernetes"
          - name: TYPHA_HEALTHENABLED
            value: "true"
          # Location of the CA bundle Typha uses to authenticate calico/node; volume mount
          - name: TYPHA_CAFILE
            value: /calico-typha-ca/typhaca.crt
          # Common name on the calico/node certificate
          - name: TYPHA_CLIENTCN
            value: calico-node
          # Location of the server certificate for Typha; volume mount
          - name: TYPHA_SERVERCERTFILE
            value: /calico-typha-certs/typha.crt
          # Location of the server certificate key for Typha; volume mount
          - name: TYPHA_SERVERKEYFILE
            value: /calico-typha-certs/typha.key
        livenessProbe:
EOF       secretName: calico-typha-certss"
deployment.apps/calico-typha created

# master node に載ると not ready のままなので 2 台に減らす
❯ k scale  --replicas=2 deploy/calico-typha -n kube-system
deployment.apps/calico-typha scaled

ubuntu@koutei:~$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: calico-typha
  namespace: kube-system
  labels:
    k8s-app: calico-typha
spec:
  ports:
    - port: 5473
      protocol: TCP
      targetPort: calico-typha
      name: calico-typha
  selector:
    k8s-app: calico-typha
EOF
service/calico-typha created

ubuntu@koutei:~$ TYPHA_CLUSTERIP=$(kubectl get svc -n kube-system calico-typha -o jsonpath='{.spec.clusterIP}')
curl https://$TYPHA_CLUSTERIP:5473 -v --cacert typhaca.crt
*   Trying 10.109.235.204:5473...
* Connected to 10.109.235.204 (10.109.235.204) port 5473 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*  CAfile: typhaca.crt
*  CApath: /etc/ssl/certs
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Request CERT (13):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Certificate (11):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS alert, bad certificate (554):
* error:14094412:SSL routines:ssl3_read_bytes:sslv3 alert bad certificate
* Closing connection 0
curl: (35) error:14094412:SSL routines:ssl3_read_bytes:sslv3 alert bad certificate
```

### 6. Install calico/node

```
ubuntu@koutei:~$ openssl req -newkey rsa:4096 \
           -keyout calico-node.key \
           -nodes \
           -out calico-node.csr \
           -subj "/CN=calico-node"
Generating a RSA private key
.................................++++
.........................++++
writing new private key to 'calico-node.key'
-----
ubuntu@koutei:~$ openssl x509 -req -in calico-node.csr \
                  -CA typhaca.crt \
                  -CAkey typhaca.key \
                  -CAcreateserial \
                  -out calico-node.crt \
                  -days 365
Signature ok
subject=CN = calico-node
Getting CA Private Key
ubuntu@koutei:~$ kubectl create secret generic -n kube-system calico-node-certs --from-file=calico-node.key --from-file=calico-node.crt
secret/calico-node-certs created

ubuntu@koutei:~$ kubectl create serviceaccount -n kube-system calico-node
serviceaccount/calico-node created
ubuntu@koutei:~$ kubectl apply -f - <<EOF
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: calico-node
rules:
  # The CNI plugin needs to get pods, nodes, and namespaces.
  - apiGroups: [""]
    resources:
      - pods
      - nodes
      - namespaces
    verbs:
      - get
  # EndpointSlices are used for Service-based network policy rule
  # enforcement.
  - apiGroups: ["discovery.k8s.io"]
    resources:
      - endpointslices
    verbs:
      - watch
      - list
  - apiGroups: [""]
    resources:
      - endpoints
      - services
    verbs:
      # Used to discover service IPs for advertisement.
      - watch
      - list
      # Used to discover Typhas.
      - get
  # Pod CIDR auto-detection on kubeadm needs access to config maps.
  - apiGroups: [""]
    resources:
      - configmaps
    verbs:
      - get
  - apiGroups: [""]
    resources:
      - nodes/status
    verbs:
      # Needed for clearing NodeNetworkUnavailable flag.
      - patch
      # Calico stores some configuration information in node annotations.
      - update
  # Watch for changes to Kubernetes NetworkPolicies.
  - apiGroups: ["networking.k8s.io"]
    resources:
      - networkpolicies
    verbs:
      - watch
      - list
  # Used by Calico for policy information.
  - apiGroups: [""]
    resources:
      - pods
      - namespaces
      - serviceaccounts
    verbs:
      - list
      - watch
EOF   - watchaffinitiesojectcalico.org"]ble by confd for route aggregation.ns.
clusterrole.rbac.authorization.k8s.io/calico-node created
ubuntu@koutei:~$ kubectl create clusterrolebinding calico-node --clusterrole=calico-node --serviceaccount=kube-system:calico-node
clusterrolebinding.rbac.authorization.k8s.io/calico-node created


```
