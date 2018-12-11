## How To Manually CRUD K8S object values in Etcd Data Store
### Requirement
   Etcd Data is key value based. It is a core component of Kubernetes which has all critical cluster info stored. As K8S Apiserver is stateless, the data in Etcd becomes paramount. This wiki to address how we can manually check values in Etcd Data store in case we lost our K8S cluster, we may still get some infor from Etcd Refer more details in [Github Etcd Guid](https://github.com/etcd-io/etcd/blob/master/Documentation/dev-guide/interacting_v3.md)

### Preparation
* Install etcdctl. As etcdctl is shipped along with etcd, we would install Etcd on the host.  Please refer official [github etcd release guide](https://github.com/etcd-io/etcd/releases)

```
ETCD_VER=v3.3.10
# choose either URL
GOOGLE_URL=https://storage.googleapis.com/etcd
GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
DOWNLOAD_URL=${GOOGLE_URL}

rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
rm -rf /tmp/etcd-download-test && mkdir -p /tmp/etcd-download-test

curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd-download-test --strip-components=1
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz

/tmp/etcd-download-test/etcd --version
ETCDCTL_API=3 /tmp/etcd-download-test/etcdctl version
```

* Prepare for CA , Certificate and Key files
 * As Etcd is secured by SSL/TLS,we need CA, Cert and key files to communicate with Etcd service. Otherwise you may get error like
 ```
 Error:  rpc error: code = 13 desc = transport: write tcp 127.0.0.1:42351->127.0.0.1:2379: write: connection reset by peer
 ```
 * Find pod name from docker ps
 ```
 # docker ps |grep etcd |grep -v pause
7872002af18a   668349ea0328  "etcd --peer-key-fil…"   7 days ago  Up 7 days   k8s_etcd_etcd-instance-cas-mt2_kube-system_1666a5cbad952bf3429406729247f6e8_3
```
 * Use docker cp to copy files out to /tmp/etcd-download-test/
 ```
 #docker cp k8s_etcd_etcd-instance-cas-mt2_kube-system_1666a5cbad952bf3429406729247f6e8_3:/etc/kubernetes/pki/etcd/ca.crt /tmp/etcd-download-test/
 #docker cp k8s_etcd_etcd-instance-cas-mt2_kube-system_1666a5cbad952bf3429406729247f6e8_3:/etc/kubernetes/pki/etcd/peer.crt /tmp/etcd-download-test/
 #docker cp k8s_etcd_etcd-instance-cas-mt2_kube-system_1666a5cbad952bf3429406729247f6e8_3:/etc/kubernetes/pki/etcd/peer.key /tmp/etcd-download-test/
 ```
 * Check Etcd endpoint status
 ```
 ./etcdctl --endpoints=127.0.0.1:2379 --cacert=./ca.crt --cert=./peer.crt --key=./peer.key endpoint status
127.0.0.1:2379, 8e9e05c52164694d, 3.1.12, 6.0 MB, true, 5, 4863422
```
 
### CRUD actions on Etcd Data
* Create 
```
export ETCDCTL_API=3   (very important)
cd /tmp/etcd-download-test/
./etcdctl --endpoints=127.0.0.1:2379 --cacert=./ca.crt --cert=./peer.crt --key=./peer.key  put henryxie testtest
OK
```

* Read
```
./etcdctl --endpoints=127.0.0.1:2379 --cacert=./ca.crt --cert=./peer.crt --key=./peer.key get henryxie
henryxie
testtest
```
* Update (Put a new value into the same key, it will update it)
```
./etcdctl --endpoints=127.0.0.1:2379 --cacert=./ca.crt --cert=./peer.crt --key=./peer.key  put henryxie barrrr
OK
./etcdctl --endpoints=127.0.0.1:2379 --cacert=./ca.crt --cert=./peer.crt --key=./peer.key get henryxie
henryxie
barrrr
```
* Delete
```
./etcdctl --endpoints=127.0.0.1:2379 --cacert=./ca.crt --cert=./peer.crt --key=./peer.key del henryxie
1
[root@instance-cas-mt2 etcd-download-test]# ./etcdctl --endpoints=127.0.0.1:2379 --cacert=./ca.crt --cert=./peer.crt --key=./peer.key get henryxie   (nothing return)
```
* Get list of all Keys in Kubernetes cluster,use grep to find your keys. ie to find all keys of livesql
```
./etcdctl --endpoints=127.0.0.1:2379 --cacert=./ca.crt --cert=./peer.crt --key=./peer.key get / --prefix --keys-only 
......
......
./etcdctl --endpoints=127.0.0.1:2379 --cacert=./ca.crt --cert=./peer.crt --key=./peer.key get / --prefix --keys-only |grep livesql
/registry/deployments/default/livesqlsb-db
/registry/deployments/default/livesqlsb-mt-deployment
/registry/ingress/default/livesqlsbroot
/registry/persistentvolumeclaims/default/livesql-pv-nfs-claim1
/registry/persistentvolumes/livesqlsb-prometheus-pv-name
/registry/persistentvolumes/livesqlsb-pv-nfs-volume1
/registry/pods/default/livesqlsb-db-56f57cd755-wldt6
/registry/pods/default/livesqlsb-mt-deployment-8895dfc8c-fsfbz
/registry/replicasets/default/livesqlsb-db-56f57cd755
/registry/replicasets/default/livesqlsb-mt-deployment-8895dfc8c
/registry/services/endpoints/default/livesqlsb-db-service
/registry/services/endpoints/default/livesqlsb-service
/registry/services/specs/default/livesqlsb-db-service
/registry/services/specs/default/livesqlsb-service
```
* Data output structure description
```
/registry/<kind type>/<namespace>/<object name>
```
We also can use "kubectl -n <namespace> get <kind type> <object name> -o json" to fetch the value
```
kubectl -n default get pod livesqlsb-db-56f57cd755-wldt6 -o json
```
