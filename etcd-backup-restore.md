## INSTALL ETCDCTL AND ETCDUTL ##
```bash
ETCD_VER=v3.6.4

# choose either URL
GOOGLE_URL=https://storage.googleapis.com/etcd
GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
DOWNLOAD_URL=${GOOGLE_URL}

rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
rm -rf /tmp/etcd-download-test && mkdir -p /tmp/etcd-download-test

curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd-download-test --strip-components=1 --no-same-owner
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz

/tmp/etcd-download-test/etcd --version
/tmp/etcd-download-test/etcdctl version
/tmp/etcd-download-test/etcdutl version
```

## EXPORT COMMON VARS
```bash
export ETCDCTL_API=3
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
```

### TAKE BACKUP:
```bash
vagrant@controlplane:~$ sudo /tmp/etcd-download-test/etcdctl --endpoints=$ETCDCTL_ENDPOINTS --cacert=$ETCDCTL_CACERT --cert=$ETCDCTL_CERT --key=$ETCDCTL_KEY snapshot save /backups/september/etcd-backup-$(date +%Y-%m-%d_%H-%M-%S).db
{"level":"info","ts":"2025-09-09T13:34:18.424964Z","caller":"snapshot/v3_snapshot.go:83","msg":"created temporary db file","path":"/backups/september/etcd-backup-2025-09-09_13-34-18.db.part"}
{"level":"info","ts":"2025-09-09T13:34:18.431399Z","logger":"client","caller":"v3@v3.6.4/maintenance.go:236","msg":"opened snapshot stream; downloading"}
{"level":"info","ts":"2025-09-09T13:34:18.441489Z","caller":"snapshot/v3_snapshot.go:96","msg":"fetching snapshot","endpoint":"https://127.0.0.1:2379"}
{"level":"info","ts":"2025-09-09T13:34:18.487681Z","logger":"client","caller":"v3@v3.6.4/maintenance.go:302","msg":"completed snapshot read; closing"}
{"level":"info","ts":"2025-09-09T13:34:18.510487Z","caller":"snapshot/v3_snapshot.go:111","msg":"fetched snapshot","endpoint":"https://127.0.0.1:2379","size":"4.0 MB","took":"85.418064ms","etcd-version":""}
{"level":"info","ts":"2025-09-09T13:34:18.510760Z","caller":"snapshot/v3_snapshot.go:121","msg":"saved","path":"/backups/september/etcd-backup-2025-09-09_13-34-18.db"}
Snapshot saved at /backups/september/etcd-backup-2025-09-09_13-34-18.db
```

## VERIFY THE SNAPSHOT STATUS
```bash
vagrant@controlplane:~$ sudo /tmp/etcd-download-test/etcdutl snapshot status /backups/september/etcd-backup-2025-09-09_13-34-18.db 
e0f36ebd, 26401, 409, 4.0 MB, 
vagrant@controlplane:~$ sudo /tmp/etcd-download-test/etcdutl --write-out="table" snapshot status /backups/september/etcd-backup-2025-09-09_13-34-18.db
+----------+----------+------------+------------+---------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE | VERSION |
+----------+----------+------------+------------+---------+
| e0f36ebd |    26401 |        409 |     4.0 MB |         |
+----------+----------+------------+------------+---------+
```

### RESTORE
```bash
vagrant@controlplane:~$ sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
vagrant@controlplane:~$ sudo mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/
vagrant@controlplane:~$ sudo mv /etc/kubernetes/manifests/kube-controller-manager.yaml /tmp/
vagrant@controlplane:~$ sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/
vagrant@controlplane:~$ sudo crictl ps
WARN[0000] runtime connect using default endpoints: [unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]. As the default settings are now deprecated, you should set the endpoint instead.
WARN[0000] image connect using default endpoints: [unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]. As the default settings are now deprecated, you should set the endpoint instead.
CONTAINER           IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID              POD
14ca5c2b3ec44       1cf5f116067c6       3 hours ago         Running             coredns             0                   3fa40f2b50437       coredns-674b8bbfcf-sc7ss
8c8f2a1f09628       1b2ea5e018dbb       3 hours ago         Running             kube-proxy          0                   9a61055611a92       kube-proxy-lm5zh
4183ac0250858       5de71980e553f       7 hours ago         Running             kube-flannel        0                   04eaae35e0e51       kube-flannel-ds-rp58w

# With the above we have stopped all the major componenets like kube-api-server, controllermanager, scheduler, etcd

# Now move/rename the etcd dir
sudo mv /var/lib/etcd /var/lib/etcd-$(date +%Y-%m-%d_%H-%M-%S)

# Restore
vagrant@controlplane:~$ sudo /tmp/etcd-download-test/etcdutl --data-dir=/var/lib/etcd snapshot restore /backups/september/etcd-backup-2025-09-09_13-34-18.db 
2025-09-09T13:45:13Z    info    snapshot/v3_snapshot.go:305     restoring snapshot      {"path": "/backups/september/etcd-backup-2025-09-09_13-34-18.db", "wal-dir": "/var/lib/etcd/member/wal", "data-dir": "/var/lib/etcd", "snap-dir": "/var/lib/etcd/member/snap", "initial-memory-map-size": 10737418240}
2025-09-09T13:45:13Z    info    bbolt   backend/backend.go:203  Opening db file (/var/lib/etcd/member/snap/db) with mode -rw------- and with options: {Timeout: 0s, NoGrowSync: false, NoFreelistSync: true, PreLoadFreelist: false, FreelistType: , ReadOnly: false, MmapFlags: 8000, InitialMmapSize: 10737418240, PageSize: 0, NoSync: false, OpenFile: 0x0, Mlock: false, Logger: 0xc000136098}
2025-09-09T13:45:13Z    info    bbolt   bbolt@v1.4.2/db.go:321  Opening bbolt db (/var/lib/etcd/member/snap/db) successfully
2025-09-09T13:45:13Z    info    schema/membership.go:138        Trimming membership information from the backend...
2025-09-09T13:45:13Z    info    bbolt   backend/backend.go:203  Opening db file (/var/lib/etcd/member/snap/db) with mode -rw------- and with options: {Timeout: 0s, NoGrowSync: false, NoFreelistSync: true, PreLoadFreelist: false, FreelistType: , ReadOnly: false, MmapFlags: 8000, InitialMmapSize: 10737418240, PageSize: 0, NoSync: false, OpenFile: 0x0, Mlock: false, Logger: 0xc000092018}
2025-09-09T13:45:13Z    info    bbolt   bbolt@v1.4.2/db.go:321  Opening bbolt db (/var/lib/etcd/member/snap/db) successfully
2025-09-09T13:45:13Z    info    membership/cluster.go:424       added member    {"cluster-id": "cdf818194e3a8c32", "local-member-id": "0", "added-peer-id": "8e9e05c52164694d", "added-peer-peer-urls": ["http://localhost:2380"], "added-peer-is-learner": false}
2025-09-09T13:45:13Z    info    bbolt   backend/backend.go:203  Opening db file (/var/lib/etcd/member/snap/db) with mode -rw------- and with options: {Timeout: 0s, NoGrowSync: false, NoFreelistSync: true, PreLoadFreelist: false, FreelistType: , ReadOnly: false, MmapFlags: 8000, InitialMmapSize: 10737418240, PageSize: 0, NoSync: false, OpenFile: 0x0, Mlock: false, Logger: 0xc0001f8018}
2025-09-09T13:45:13Z    info    bbolt   bbolt@v1.4.2/db.go:321  Opening bbolt db (/var/lib/etcd/member/snap/db) successfully
2025-09-09T13:45:13Z    info    snapshot/v3_snapshot.go:333     restored snapshot       {"path": "/backups/september/etcd-backup-2025-09-09_13-34-18.db", "wal-dir": "/var/lib/etcd/member/wal", "data-dir": "/var/lib/etcd", "snap-dir": "/var/lib/etcd/member/snap", "initial-memory-map-size": 10737418240}

vagrant@controlplane:~$ sudo mv /tmp/etcd.yaml /etc/kubernetes/manifests/
vagrant@controlplane:~$ sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
vagrant@controlplane:~$ sudo mv /tmp/kube-scheduler.yaml /etc/kubernetes/manifests/
vagrant@controlplane:~$ sudo mv /tmp/kube-controller-manager.yaml /etc/kubernetes/manifests/
vagrant@controlplane:~$ sudo crictl ps
WARN[0000] runtime connect using default endpoints: [unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]. As the default settings are now deprecated, you should set the endpoint instead.
WARN[0000] image connect using default endpoints: [unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]. As the default settings are now deprecated, you should set the endpoint instead.
CONTAINER           IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID              POD
878ffdee87c5c       358ab71c1a1ea       10 seconds ago      Running             kube-controller-manager   0                   36f1556bd8310       kube-controller-manager-controlplane
a07aa929e6430       ab4ad8a84c3c6       15 seconds ago      Running             kube-scheduler            0                   cf3e9836ab40c       kube-scheduler-controlplane
aad81150a5cd2       1f41885d0a911       22 seconds ago      Running             kube-apiserver            0                   d69079f82217a       kube-apiserver-controlplane
a7b6556dff287       499038711c081       30 seconds ago      Running             etcd                      0                   b6a107ef9f2fd       etcd-controlplane
14ca5c2b3ec44       1cf5f116067c6       3 hours ago         Running             coredns                   0                   3fa40f2b50437       coredns-674b8bbfcf-sc7ss
8c8f2a1f09628       1b2ea5e018dbb       4 hours ago         Running             kube-proxy                0                   9a61055611a92       kube-proxy-lm5zh
4183ac0250858       5de71980e553f       7 hours ago         Running             kube-flannel              0                   04eaae35e0e51       kube-flannel-ds-rp58w

```
