# cka-practice
- Certified Kubernetes Admin Study
- K8S Documentation
  - https://kubernetes.io/docs/home/
## 외부 시스템에서 K8S Cluster 접근하기
- K8S Documentation > ``access multiple cluster`` 검색 > ``다중 클러스터 접근 구성 | Kubernetes`` 선택
  - https://kubernetes.io/ko/docs/tasks/access-application-cluster/configure-access-multiple-clusters/
  - Step 01: 외부 시스템에 kubectl 설치하기
    - https://kubernetes.io/ko/docs/tasks/tools/install-kubectl-linux/
  - Step 02: TODO
* * *  
# 01 ETCD Backup & Restore
## 검색 방법
- 검색어: ``etcd backup restore``
  - ``Operating etcd clusters for Kubernetes | Kubernetes`` 선택
  - https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster
## Problem: etcd backup & restore
- 작업 시스템: k8s-master   
  First, create a snapshot of the existing ``etcd`` instance running at ``http://127.0.0.1:2379``, saving the snapshot to ``/data/etcd-snapshot.db``. Next, restore an existing, previous snapshot located at ``/data/etcd-snapshot-previous.db``.      
   
  The following TLS ``certificates/key`` are supplied for connecting to the server with etcdctl:
    - CA certificate: ``/etc/kubernetes/pki/etcd/ca/ca.crt``
    - Client certificate: ``/etc/kubernetes/pki/etcd/server.crt``
    - Client key: ``/etc/kubernetes/pki/etcd/server.key``
## ETCD Backup과 Restore에 대한 소개
- ETCD
  - API에 의해서 동작되는 K8S의 모든 운영 정보가 담겨져 있는 저장소 역할
    - ``Key:Value`` 타입의 저장소
  - 하나의 Pod 형태로 동작되고 있음
  - 실제로는 메모리 공간에서 동작하지만, ``/var/lib/etcd``에 데이터베이스 형태로 함께 저장
- ETCD Backup
  - 현재의 ETCD 정보를 Snapshot을 만드는 개념
  - 예: ``/data/etcd-snapshot.db``
- ETCD Restore
  - 저장된 ETCD Snapshot을 이용해서 Restore
    - Step01: 저장된 ETCD Snapshot 파일(예:``/data/etcd-snapshot-previous.db``)을 ``/var/lib/etcd``이 아닌 다른 경로에 풀어줌      
    - Step02: K8S cluster가 새로운 경로의 ETCD를 바라볼 수 있도록 ``Configuration``을 변경
    - Step03: ETCD Pod는 Static Pod이므로, ``Configuration``을 변경하면 Auto Restart
- ETCD Backup & Restore Diagram   
  ![etcd_backup_recovery](./images/etcd_backup_recovery.png)
## Answer
```bash
# Step 01: 작업 시스템에 접속
$ ssh k8s-master
# Step 02: 현재의 K8S current context 정보 얻기
$ kubectl config get-contexts
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         example@kubernetes            kubernetes   kubernetes-admin   product
          ingress-admin@kubernetes      kubernetes   kubernetes-admin   ingress-nginx
          kubernetes-admin@kubernetes   kubernetes   kubernetes-admin   default
$ kubectl config current-context
example@kubernetes
# Step 03: tree 명령어를 이용해 디렉토리 구조를 Display
$ sudo tree /var/lib/etcd
/var/lib/etcd
└── member
    ├── snap
    │   ├── 0000000000000032-000000000007a152.snap
    │   ├── 0000000000000033-000000000007c863.snap
    │   ├── 0000000000000033-000000000007ef74.snap
    │   ├── 0000000000000034-0000000000081685.snap
    │   ├── 0000000000000034-0000000000083d96.snap
    │   └── db
    └── wal
        ├── 0000000000000002-000000000002c07d.wal
        ├── 0000000000000003-0000000000041ac1.wal
        ├── 0000000000000004-0000000000056427.wal
        ├── 0000000000000005-000000000006cab3.wal
        ├── 0000000000000006-0000000000082c74.wal
        └── 1.tmp

3 directories, 12 files
# Step 04: etcdctl 설치 (없다면)
$ wget https://github.com/etcd-io/etcd/releases/download/v3.5.4/etcd-v3.5.4-linux-amd64.tar.gz
$ tar -zxvf etcd-v3.5.4-linux-amd64.tar.gz
$ cd etcd-v3.5.4-linux-amd64/
~/etcd-v3.5.4-linux-amd64$ sudo mv etcd  etcdctl  etcdutl /usr/local/bin
$ etcdctl version
etcdctl version: 3.5.4
API version: 3.5
# Step 05: Snapshot using etcdctl options
$ ETCDCTL_API=3 sudo etcdctl --endpoints=https://127.0.0.1:2379   --cacert=/etc/kubernetes/pki/etcd/ca.crt   --cert=/etc/kubernetes/pki/etcd/server.crt   --key=/etc/kubernetes/pki/etcd/server.key   snapshot save /data/etcd-snapshot.db
{"level":"info","ts":1661681002.3225548,"caller":"snapshot/v3_snapshot.go:119","msg":"created temporary db file","path":"/data/etcd-snapshot.db.part"}
{"level":"info","ts":"2022-08-28T19:03:22.330+0900","caller":"clientv3/maintenance.go:200","msg":"opened snapshot stream; downloading"}
{"level":"info","ts":1661681002.3307705,"caller":"snapshot/v3_snapshot.go:127","msg":"fetching snapshot","endpoint":"https://127.0.0.1:2379"}
{"level":"info","ts":"2022-08-28T19:03:22.397+0900","caller":"clientv3/maintenance.go:208","msg":"completed snapshot read; closing"}
{"level":"info","ts":1661681002.4073474,"caller":"snapshot/v3_snapshot.go:142","msg":"fetched snapshot","endpoint":"https://127.0.0.1:2379","size":"4.8 MB","took":0.08449835}
{"level":"info","ts":1661681002.40754,"caller":"snapshot/v3_snapshot.go:152","msg":"saved","path":"/data/etcd-snapshot.db"}
Snapshot saved at /data/etcd-snapshot.db
$ ll /data/etcd-snapshot.db
-rw------- 1 root root 4808736  8월 28 19:03 /data/etcd-snapshot.db
# Step 05-1: 테스트를 위해 새로운 Pod 생성
$ kubectl run testpod --image=nginx
pod/testpod created
$ kubectl get pods
NAME                          READY   STATUS              RESTARTS       AGE
testpod                       0/1     ContainerCreating   0              4s
# Step 06: Restoring an etcd cluster
#  - /data/etcd-snapshot-previous.db을 읽어서 /var/lib/etcd-previous에 압축을 품
$ sudo cp /data/etcd-snapshot.db /data/etcd-snapshot-previous.db
$ ETCDCTL_API=3 sudo etcdctl --data-dir /var/lib/etcd-previous snapshot restore /data/etcd-snapshot-previous.db
{"level":"info","ts":1661681885.7301135,"caller":"snapshot/v3_snapshot.go:296","msg":"restoring snapshot","path":"/data/etcd-snapshot-previous.db","wal-dir":"/var/lib/etcd-previous/member/wal","data-dir":"/var/lib/etcd-previous","snap-dir":"/var/lib/etcd-previous/member/snap"}
{"level":"info","ts":1661681885.7666476,"caller":"mvcc/kvstore.go:388","msg":"restored last compact revision","meta-bucket-name":"meta","meta-bucket-name-key":"finishedCompactRev","restored-compact-revision":477763}
{"level":"info","ts":1661681885.775317,"caller":"membership/cluster.go:392","msg":"added member","cluster-id":"cdf818194e3a8c32","local-member-id":"0","added-peer-id":"8e9e05c52164694d","added-peer-peer-urls":["http://localhost:2380"]}
{"level":"info","ts":1661681885.781045,"caller":"snapshot/v3_snapshot.go:309","msg":"restored snapshot","path":"/data/etcd-snapshot-previous.db","wal-dir":"/var/lib/etcd-previous/member/wal","data-dir":"/var/lib/etcd-previous","snap-dir":"/var/lib/etcd-previous/member/snap"}
gusami@master:~$ sudo tree /var/lib/etcd-previous/
/var/lib/etcd-previous/
└── member
    ├── snap
    │   ├── 0000000000000001-0000000000000001.snap
    │   └── db
    └── wal
        └── 0000000000000000-0000000000000000.wal
# Step 05: etcd Pod에게 바뀐 위치를 알려줌
# Step 05-1: etcd pod 확인. master node에서 static pod 형태로 동작 중임
$ kubectl get pods -o wide -n kube-system
NAME                              READY   STATUS    RESTARTS          AGE    IP          NODE       NOMINATED NODE   READINESS GATES
coredns-78fcd69978-nz4fr          1/1     Running   49 (105d ago)     288d   10.32.0.3   master     <none>           <none>
coredns-78fcd69978-vsnzh          1/1     Running   49 (105d ago)     288d   10.32.0.2   master     <none>           <none>
etcd-master                       1/1     Running   50 (105d ago)     288d   10.0.1.4    master     <none>           <none>
kube-apiserver-master             1/1     Running   50 (105d ago)     288d   10.0.1.4    master     <none>           <none>
kube-controller-manager-master    1/1     Running   50 (105d ago)     288d   10.0.1.4    master     <none>           <none>
kube-proxy-2pm78                  1/1     Running   30 (105d ago)     212d   10.0.1.7    worker-3   <none>           <none>
kube-proxy-s9cp2                  1/1     Running   47 (105d ago)     288d   10.0.1.5    worker-1   <none>           <none>
kube-proxy-vscnf                  1/1     Running   50 (105d ago)     288d   10.0.1.4    master     <none>           <none>
kube-proxy-wsc4f                  1/1     Running   46 (105d ago)     288d   10.0.1.6    worker-2   <none>           <none>
kube-scheduler-master             1/1     Running   50 (105d ago)     288d   10.0.1.4    master     <none>           <none>
metrics-server-774b56d589-bmhx2   1/1     Running   1 (105d ago)      106d   10.44.0.2   worker-1   <none>           <none>
weave-net-9s8mx                   2/2     Running   64 (3h38m ago)    212d   10.0.1.7    worker-3   <none>           <none>
weave-net-kzxnd                   2/2     Running   100 (105d ago)    288d   10.0.1.6    worker-2   <none>           <none>
weave-net-tbcxg                   2/2     Running   100 (3h38m ago)   288d   10.0.1.4    master     <none>           <none>
weave-net-zvqg4                   2/2     Running   96 (105d ago)     288d   10.0.1.5    worker-1   <none>           <none>
# Step 05-2
#  - static pod의 yaml 파일 위치로 이동
#    - /etc/kubernetes/manifests/
#  - volume mount되는 hostPath의 path를 수정한 후, 저장
#    - path: /var/lib/etcd-previous
$ cd /etc/kubernetes/manifests/
/etc/kubernetes/manifests$ ls
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
/etc/kubernetes/manifests$ sudo vi etcd.yaml
.....
volumeMounts:
    - mountPath: /var/lib/etcd
      name: etcd-data
    - mountPath: /etc/kubernetes/pki/etcd
      name: etcd-certs
  hostNetwork: true
  priorityClassName: system-node-critical
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcd-previous
      type: DirectoryOrCreate
    name: etcd-data
status: {}
# docker ps 명령어를 이용해서 etcd가 재시작했는지 체크. 1분 정도 걸림
#   - Up 상태로 바뀌어야 함
$ sudo docker ps -a | grep etcd
8aa4c20dd37e   004811815584           "etcd --advertise-cl…"   13 seconds ago   Up 12 seconds                          k8s_etcd_etcd-master_kube-system_a1d9c38bb8a1f3b393b7958e55f554e8_0
683e64640a2f   k8s.gcr.io/pause:3.5   "/pause"                 13 seconds ago   Up 13 seconds                          k8s_POD_etcd-master_kube-system_a1d9c38bb8a1f3b393b7958e55f554e8_1
# test pod가 없는 것을 확인
$ kubectl get pods
NAME                          READY   STATUS    RESTARTS       AGE

```
23:35
* * *
# 02. Pod 생성하기
## 검색 방법
## Problem: Pod 생성하기
