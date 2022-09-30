# OpenShift 클러스터 상 애플리케이션 백업 및 복구 (네임스페이스 단위)


## 0. 사전 준비
OpenShift 클러스터에서 Operator Hub를 통해 ODF Operator 설치 후 기본 설정으로 스토리지 시스템을 생성합니다. 


## 1. 네임스페이스 백업용 오브젝트 버킷 클레임 생성
아래와 비슷한 설정으로 개체 버킷 클레임(object bucket claim)을 생성합니다.

```yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: namespace-bucket-claim
  namespace: openshift-storage
spec:
  generateBucketName: namespace-backup-bucket
  storageClassName: openshift-storage.noobaa.io
  additionalConfig:
    bucketclass: noobaa-default-bucket-class
```

```yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: acm-bucket-claim
  namespace: openshift-storage
spec:
  generateBucketName: acm-backup-bucket
  storageClassName: openshift-storage.noobaa.io
  additionalConfig:
    bucketclass: noobaa-default-bucket-class
```



## 2. ODF로 OADP 설치 및 구성

1. 오퍼레이터 허브를 통해 OADP 오퍼레이터 설치

2. OADP용 시크릿을 생성합니다. 

```bash
NOOBAA_ACCESS_KEY=$(kubectl get secret noobaa-admin -n openshift-storage -o json | jq -r '.data.AWS_ACCESS_KEY_ID|@base64d')
NOOBAA_SECRET_KEY=$(kubectl get secret noobaa-admin -n openshift-storage -o json | jq -r '.data.AWS_SECRET_ACCESS_KEY|@base64d')
```

```bash
cat << EOF >> credentials-velero
[default]
aws_access_key_id = ${NOOBAA_ACCESS_KEY}
aws_secret_access_key = ${NOOBAA_SECRET_KEY}
EOF
```

```bash
oc create secret generic cloud-credentials --from-file cloud=credentials-velero -n openshift-adp
```




## 3. 네임스페이스 백업용 DPA 생성

```yaml
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: project-backup
  namespace: openshift-adp
spec:
  backupLocations:
    - velero:
        provider: aws
        default: true
        config:
          profile: default
          region: noobaa
          s3Url: https://s3.openshift-storage.svc:443
          s3ForcePathStyle: "true"
          insecureSkipTLSVerify: "true"
        objectStorage:
          bucket: ## <<버킷 이름>>
          prefix: hub
        credential:
          key: cloud
          name: cloud-credentials
  configuration:
    featureFlags:
      - EnableCSI
    velero:
      defaultPlugins:
        - openshift
        - aws
        - csi
      podConfig:
        resourceAllocations:
          limits:
            cpu: "2"
            memory: 1Gi
          requests:
            cpu: 500m
            memory: 256Mi
```



## 4. 볼륨 스냅샷 클래스 수정

=> 스토리지 시스템 생성 시 csi 지원 스토리지로 생성하고 DPA에서 CSI enable 해주었다면 필요없음

```
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: <volume_snapshot_class_name>
  labels:
    velero.io/csi-volumesnapshot-class: "true"
driver: <csi_driver>
deletionPolicy: Retain
```





## 5. 백업 생성

### 5-1. 백업

`Backup`CR를 생성하여 Kubernetes 이미지, 내부 이미지 및 영구 볼륨(PV)을 백업합니다.
// 또는 콘솔에서 installed operator -> OADP operator -> backup 생성 -> IncludedNamespaces에서 백업할 네임스페이스 이름을 입력할 수도 있습니다.

```yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: <backup> # 백업의 이름
  labels:
    velero.io/storage-location: default
  namespace: openshift-adp
spec:
  hooks: {}
  includedNamespaces:
  - <namespace> # 백업할 네임스페이스 목록
  storageLocation: <velero-sample-1> # DPA 생성하면서 같이 생성된 backupStorageLocationsCR의 이름을 지정
  ttl: 720h0m0s
```



### 5-2. 백업 대신 예약 생성

5-1이 바로 백업을 진행하는 단계라면, 5-2는 바로 백업을 진행하는 대신 백업을 예약합니다. 

```yaml
$ cat << EOF | oc apply -f -
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: <schedule>
  namespace: openshift-adp
spec:
  schedule: 0 7 * * * 
  template:
    hooks: {}
    includedNamespaces:
    - <namespace> 
    storageLocation: <velero-sample-1> 
    ttl: 720h0m0s
EOF
```
아래 명령어를 통해 백업 예약이 생성되었는지 확인합니다.
```bash
oc get schedule -n openshift-adp <schedule> -o jsonpath='{.status.phase}'
```


## 6. 복원
애플리케이션 파드나 프로젝트 삭제 후, 아래와 같은 Restore CR을 생성하여 복구합니다. 
PVC가 붙은 애플리케이션이 있다면 restorePVs 값을 true로 주어야합니다.

```yaml
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: <restore>
  namespace: openshift-adp
spec:
  backupName: <backup> 
  excludedResources:
  - nodes
  - events
  - events.events.k8s.io
  - backups.velero.io
  - restores.velero.io
  - resticrepositories.velero.io
  restorePVs: true
```



- 백업 작업 전후에 명령을 실행하는 [백업 후크](https://docs.openshift.com/container-platform/4.10/backup_and_restore/application_backup_and_restore/backing_up_and_restoring/backing-up-applications.html#oadp-creating-backup-hooks_backing-up-applications) 를 생성할 수 있습니다 .

- `Backup` CR 대신 [`Schedule`CR](https://docs.openshift.com/container-platform/4.10/backup_and_restore/application_backup_and_restore/backing_up_and_restoring/backing-up-applications.html#oadp-scheduling-backups_backing-up-applications) 을 만들어 백업을 예약할 수 있습니다.



