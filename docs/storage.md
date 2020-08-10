
Create an CephObjectStore, erasure encoded StorageClass with a reclaim policy of delete, ObjectBucketClaim and a CephObjectStoreUser
```
kubectl create -f https://github.com/rook/rook/raw/master/cluster/examples/kubernetes/ceph/object-ec.yaml
kubectl create -f https://github.com/rook/rook/raw/master/cluster/examples/kubernetes/ceph/storageclass-bucket-delete.yaml
kubectl create -f https://github.com/rook/rook/raw/master/cluster/examples/kubernetes/ceph/object-bucket-claim-delete.yaml
kubectl create -f https://github.com/rook/rook/raw/master/cluster/examples/kubernetes/ceph/object-user.yaml
```
Get the s3 details as seen inside the cluster
```
(
    echo export AWS_ENDPOINT=$(kubectl -n rook-ceph get service/rook-ceph-rgw-my-store -o jsonpath='{range .spec}{@.clusterIP}{":"}{@.ports[0].port}{"\n"}{end}')
    echo export AWS_HOST=$(kubectl -n default get cm ceph-delete-bucket -o jsonpath='{.data.BUCKET_HOST}')
    echo export AWS_ACCESS_KEY_ID=$(kubectl -n default get secret ceph-delete-bucket  -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 --decode)
    echo export AWS_SECRET_ACCESS_KEY=$(kubectl -n default get secret ceph-delete-bucket  -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 --decode)
)
```

To test we can just run a container in the foreground

```
kubectl run --interactive --tty --rm --image=centos:8 centos8
```

```
[root@centos8 /]# yum install -y epel-release
[root@centos8 /]# yum install -y s3cmd
[root@centos8 /]# export AWS_ENDPOINT=10.110.106.178:80
[root@centos8 /]# export AWS_HOST=rook-ceph-rgw-my-store.rook-ceph
[root@centos8 /]# export AWS_ACCESS_KEY_ID=MID8lJNHGBV6P8YTVREW5HY
[root@centos8 /]# export AWS_SECRET_ACCESS_KEY=6gQw11E7wTLxOHDwnXDl8WAXBYh1WyAUOVM9SnbrUN
[root@centos8 /]# echo "Hello Rook" > /tmp/rookObj
[root@centos8 /]# s3cmd --host=${AWS_HOST} --no-ssl ls
2020-08-10 22:31  s3://ceph-bkt-4ccdb462-d8fd-4540-8505-ecd589932a14
[root@centos8 /]# s3cmd put /tmp/rookObj --no-ssl --host=${AWS_HOST} --host-bucket= s3://ceph-bkt-4ccdb462-d8fd-4540-8505-ecd589932a14
upload: '/tmp/rookObj' -> 's3://ceph-bkt-4ccdb462-d8fd-4540-8505-ecd589932a14/rookObj'  [1 of 1]
 11 of 11   100% in    0s   132.64 B/s  done
[root@centos8 /]# s3cmd get --no-ssl --host=${AWS_HOST} --host-bucket= s3://ceph-bkt-4ccdb462-d8fd-4540-8505-ecd589932a14/rookObj
download: 's3://ceph-bkt-4ccdb462-d8fd-4540-8505-ecd589932a14/rookObj' -> './rookObj'  [1 of 1]
 11 of 11   100% in    0s   258.98 B/s  done
[root@centos8 /]# cat /rookObj
Hello Rook
```


