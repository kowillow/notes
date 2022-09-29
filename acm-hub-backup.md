# RHACM Hub 백업 및 복원



## 1. 사전 준비

### 1-1. 액티브 허브/패시브 허브 공통

아래와 같은 오퍼레이터가 설치되어 있어야 합니다.
- ACM 2.5.x 이상



ACM 허브에서 백업 컴포넌트를 활성화합니다. (Active, Passive 모두)

```bash
oc patch MultiClusterHub multiclusterhub -n open-cluster-management --type=json -p='[{"op": "add", "path": "/spec/overrides/components/-","value":{"name":"cluster-backup","enabled":true}}]'
```



### 1-2. 패시브 허브 준비

- 액티브 허브와 동일한 네임스페이스에 Multicluster engine, ACM, ACM 관련 오퍼레이터 설치(Gitops 등)
- ODF 4.10 설치 및 스토리지 시스템 구성
- 오브젝트 버킷 클레임 생성



 DataProtectionApplication용 시크릿을 생성하지 않았다면 지금 생성합니다. 그리고 액티브 허브에도 시크릿을 생성합니다.

```bash
$ oc config use-context c1
$ NOOBAA_ACCESS_KEY=$(kubectl get secret noobaa-admin -n openshift-storage -o json | jq -r '.data.AWS_ACCESS_KEY_ID|@base64d')
$ NOOBAA_SECRET_KEY=$(kubectl get secret noobaa-admin -n openshift-storage -o json | jq -r '.data.AWS_SECRET_ACCESS_KEY|@base64d')
```

```bash
cat << EOF >> credentials-velero
[default]
aws_access_key_id = ${NOOBAA_ACCESS_KEY}
aws_secret_access_key = ${NOOBAA_SECRET_KEY}
EOF
```

```bash
$ oc create secret generic cloud-credentials --from-file cloud=credentials-velero -n openshift-adp
$ oc config use-context hub
$ oc create secret generic cloud-credentials --from-file cloud=credentials-velero -n openshift-adp
```



## 2. 허브 백업 생성 및 예약

### 2-2. 예비 허브(cluster-2)의 ODF에서 오브젝트 버킷 클레임 생성

> 클러스터 백업 및 복원 리소스는 OADP 운영자가 설치된 동일한 네임스페이스에 생성되어야 합니다... (버킷 위치는 별도?)

#### ~예비 허브~에 만들어줍니다.

Noobaa 스토리지 클래스로 오브젝트 버킷 클레임 생성

![bucket-claim](img/bucket-claim.png)



### 2-3. 액티브 허브에서 DPA 생성

생성된 버킷 정보에서 **Bucket Name**(`acm-bucket-c7a5a832-f110-4af2-893b-1e7fa10f1cda` 같은 형식)을 아래 DPA YAML에 입력 

```yaml
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: acm-bak
  namespace: openshift-adp ## if oadp operator is not installed automatically, use this.
spec:
  backupLocations:
    - velero:
        provider: aws
        default: true
        config:
          profile: default
          region: noobaa
          s3Url: ## passive hub의 s3 external URL e.g.https://s3-openshift-storage.apps.ocp-dev3.poc.cloud
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



### 2-4. 백업 스케줄 생성

`backupschedule` 리소스를 생성하면 허브 백업 일정이 활성화됩니다. **OADP 오퍼레이터가 설치된 네임스페이스에 BackupSchedule을 생성**합니다.

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: BackupSchedule
metadata:
  name: schedule-acm
  namespace: openshift-adp
spec:
  veleroSchedule: 0 */2 * * *  # 2시간마다 백업 생성
  veleroTtl: 120 # 120시간 후 예약된 백업 삭제; 선택 사항, 지정하지 않으면 velero에서 설정한 최대 기본값인 720h 사용
```



이렇게 생성한 backupschedule은 ACM HUB의 리소스를 백업하는 여러 개의 schedule을 생성합니다.

```bash
$ oc get schedules -A | grep acm
openshift-adp   acm-credentials-cluster-schedule   2m44s
openshift-adp   acm-credentials-hive-schedule      2m44s
openshift-adp   acm-credentials-schedule           2m44s
openshift-adp   acm-managed-clusters-schedule      2m44s
openshift-adp   acm-resources-generic-schedule     2m44s
openshift-adp   acm-resources-schedule             2m44s
openshift-adp   acm-validation-policy-schedule     2m44s
```

- credential 백업은 3가지 credential에 대한 백업 (Hive, ACM, 유저가 직접 생성한 credential) 포함
- resource 백업은 ACM 리소스 백업 1개, generic 리소스 백업 1개 포함
- managed 백업은 **허브-매니지드 클러스터 간 연결을 활성화** 시키는 리소스만 포함. 이 백업을 복구할 경우, 복구받은 허브 클러스터가 매니지드 클러스터에 대한 연결을 가져오면서 Active Hub가 됨 (활성화 데이터)

새로운 허브 클러스터에 credential 백업과 resource 백업을 복구 시키면 Hive API로 생성한 매니지드 클러스터들만 Detached 상태로 보이게 됩니다.  Import 과정을 통해 가져온 클러스터들은 Passive Hub에서 보이지 않다가, managed 백업까지 복구시키면 *pending import* 상태로 새 Hub에서 보이게 됩니다. 새로운 허브에서 보이기는 하지만 여전히 원래의 허브에 연결되어있기 때문에 새 클러스터에 수동으로 다시 연결해야 합니다. (Hive API를 사용하여 생성한 매니지드 클러스터만 자동으로 새 허브 클러스터와 연결됨)



------

## (3. Active-Passive 구성)

- Active 허브: 클러스터를 관리하고  `BackupSchedule`에 정의된 시간 간격으로 리소스를 백업
- Passive 허브: 최신 백업을 지속적으로 검색하고 Passive 데이터를 복원. 기본 허브 클러스터가 다운되면 Activation 데이터를 복원하여 Active 허브가 됩니다. 



Active 허브 클러스터가 백업한 데이터는 다음과 같은 시나리오로 복원할 수 있습니다.

- 시나리오1: Active 허브가 백업된 스냅샷으로 복원

- 시나리오2: Passive 허브가 1회성 복원

- 시나리오3: Passive 허브가 지속적으로 패시브 데이터 복원 => 필요 시 액티브 데이터 복원하여 연결 활성화



<img src="img/active_passive_config_design.png" alt="능동 수동 구성 다이어그램" style="zoom:33%;" />

Passive 허브 클러스터는 매니지드 클러스터 활성화 데이터(매니지드 클러스터를 Passive 허브 클러스터로 이동)를 제외하고, Passive 데이터를 복원합니다.



## (4. 허브 백업을 사용해서 복구하는 방법)

### 4-1. 복구 방법 설명

- 허브 클러스터에서 `restore` 리소스를 만든 후 다음 명령을 실행하여 복원 작업의 상태를 확인할 수 있습니다. 

```yaml
# 참고용, activation data restore 포함되어있음
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Restore
metadata:
  name: restore-acm
spec:
  veleroManagedClustersBackupName: latest
  veleroCredentialsBackupName: latest
  veleroResourcesBackupName: latest
```

```bash
oc get restore -n openshift-adp
```

**주의!** : 원본 허브가 살아있는 상태에서 veleroManagedClustersBackupName을 skip하지 않고 복원하면, 
원본 허브는 매니지드 허브를 다시 가져오려고 시도합니다. 

**Note**: `restore` 리소스는 한번만 실행됩니다. 동일한 복구 작업을 다시 실행하고 싶으면 동일한 spec으로 `restore` 리소스를 다시 생성해야 합니다.



`restore.cluster.open-cluster-management.io` 리소스는 3가지 유형의 백업을 모두 복원하는 데 사용됩니다. 그러나 특정 유형의 백업만 설치하도록 선택할 수도 있습니다. 

- `veleroManagedClustersBackupName` 은 매니지드 클러스터 활성화 옵션

- `veleroCredentialsBackupName` 은 유저 credential 복원 옵션

- `veleroResourcesBackupName` 은 허브 클러스터 리소스(`Applications`, `Policy` 같은 수동 데이터) 복원 옵션

  위 3가지 속성에 대한 유효한 값은 다음과 같습니다.

  - `latest` - 최신 백업 파일 복원
  - `skip` - 이 백업 파일은 복원하지 않음
  - `<backup_name>` - 특정한 백업 파일을 지정하여 복원

위와 같이 생성한 `restore.cluster.open-cluster-management.io` 리소스는  `restore.velero.io` 리소스를 생성합니다. (skip된 백업 파일에 대해서는 생성되지 않음)



### 4-2. 복구 

#### 활성화 리소스 복원

다른 데이터는 Passive 리소스 복원을 통해 Passive 허브 클러스터에 이미 복원된 것으로 가정합니다. 활성화 리소스를 복원해서, 복원 작업을 진행하는 허브 클러스터가 매니지드 클러스터를 관리하기 시작합니다.

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Restore
metadata:
  name: restore-acm-passive-activate
spec:
  cleanupBeforeRestore: CleanupRestored
  veleroManagedClustersBackupName: latest
  veleroCredentialsBackupName: skip
  veleroResourcesBackupName: skip
```



#### 패시브 리소스 복원 (1회성)

Passive 데이터는 매니지드 클러스터와 허브 클러스터 간의 연결을 활성화하지 않는, 시크릿, ConfigMaps, 애플리케이션, 정책 및 매니지드 클러스터 CRD와 같은 백업 데이터입니다. 

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Restore
metadata:
  name: restore-acm-passive
spec:
  veleroManagedClustersBackupName: skip
  veleroCredentialsBackupName: latest
  veleroResourcesBackupName: latest
```



나머지 복원 방법은 5에서 작성합니다.



### 4-3. 복원하기 전에 허브 클러스터 Clean

이전에 복원 작업으로 생성된 리소스가 있으면, 이번 복원 작업에서 해당 리소스는 skip하고 진행하게 됩니다.

예비 허브 클러스터를 사용한 적이 있고 복원을 두 번 이상 했다면, Passive Hub로 사용하려면 Clean하고 사용하는 것이 좋습니다.

`cleanupBeforeRestore` 옵션을 지정해서 `restore.cluster.open-cluster-management.io` 리소스를 생성하면 Velero 복원이 시작되기 전에 허브 클러스터를 정리하여 복원을 준비하는 일련의 단계를 실행합니다. 옵션값은 다음과 같이 설정할 수 있습니다.

- `None`: 정리할 필요가 없습니다. Velero 복원을 시작하기만 하면 됩니다. 완전히 새로운 허브 클러스터에서 Restore 하는 경우 사용됩니다.
- `CleanupRestored`: 이전에 ACM 복원으로 생성된 모든 리소스를 정리합니다. `CleanupAll` 보다 이 속성을 사용하는 것이 좋습니다 
- `CleanupAll`: 복원 작업의 결과로 리소스가 생성되지 않은 경우에도 ACM 백업의 일부일 수 있는 허브 클러스터의 모든 리소스를 정리합니다. 정리가 필요한 허브 클러스터에 추가 콘텐츠가 생성되었을 때 사용됩니다. 이 옵션은 백업이 만든 리소스 뿐만 아니라 사용자가 만든 허브 클러스터의 리소스까지 정리하므로 주의해서 사용하십시오. 되도록  `CleanupRestored` 옵션을 사용하고, Restore를 실행하는 허브 클러스터가 DR용 Passive 클러스터로 지정된 경우 허브 클러스터 콘텐츠를 수동으로 업데이트하지 않는 것이 좋습니다 . `CleanupAll`옵션은 마지막 수단으로 사용하십시오 .

> ​       **참고**
>
> - 복원된 백업에 리소스가 없는 경우 Velero는 Velero 복원 리소스에 대해 PartialFailed 상태를 설정합니다. 즉, 생성된 restore.velero.io 리소스에 상응하는 해당 백업이 비어 있으면 리소스를 복원하지 않고, restore.cluster.open-cluster-management.io 리소스가 PartiallyFailed 상태가 될 수 있습니다.
> - syncRestoreWithNewBackups:true를 사용하면 새 백업을 확인해서 패시브 데이터를 계속 복원하지만, 그렇지 않으면 Restore는 한번만 실행됩니다. 따라서 해당 옵션이 없는 경우 동일한 허브 클러스터에서 다른 복원 작업을 실행하려면 새 restore.cluster.open-cluster-management.io 리소스를 다시 생성해야 합니다.
> - `restore.cluster.open-cluster-management.io` 리소스는 여러개 생성할 수 있지만 한번에 하나만 활성화됩니다.



----

## 5. 허브 백업 파일을 Passive 허브에서 복구

### 5-1. Passive 허브에서 DPA 생성

```yaml
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: remote-acm-bak
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



### 5-2. 복구

- Passive 데이터 복구

Passive 데이터만 복구합니다. Passive 허브 클러스터 콘솔에서 매니지드 클러스터들이 Detached로 보이는 것을 확인합니다.

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Restore
metadata:
  name: restore-acm-passive
spec:
  cleanupBeforeRestore: CleanupRestored
  veleroManagedClustersBackupName: skip
  veleroCredentialsBackupName: latest
  veleroResourcesBackupName: latest
```



- Passive Hub 싱크 유지

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Restore
metadata:
  name: restore-acm-passive-sync
spec:
  syncRestoreWithNewBackups: true # 새 백업을 사용할 수 있을 때 다시 복원
  restoreSyncInterval: 10m # 10분마다 새 백업 확인
  cleanupBeforeRestore: CleanupRestored
  veleroManagedClustersBackupName: skip
  veleroCredentialsBackupName: latest
  veleroResourcesBackupName: latest
```



원본 Active Hub를 내리고 (multicluster hub 리소스 삭제)

아래 Restore를 통해 Activation 데이터까지 복구합니다. Passive Hub가 연결을 가져가게 됩니다. 

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Restore
metadata:
  name: restore-acm-all
spec:
  cleanupBeforeRestore: CleanupRestored
  veleroManagedClustersBackupName: latest
  veleroCredentialsBackupName: latest
  veleroResourcesBackupName: latest
```



복원 작업의 상태는 아래와 같이 확인합니다.

```bash
$ oc describe -n openshift-adp <restore-name>
Spec:
  Cleanup Before Restore:               CleanupRestored
  Restore Sync Interval:                4m
  Sync Restore With New Backups:        true
  Velero Credentials Backup Name:       latest
  Velero Managed Clusters Backup Name:  skip
  Velero Resources Backup Name:         latest
Status:
  Last Message:                     Velero restores have run to completion, restore will continue to sync with new backups
  Phase:                            Enabled
  Velero Credentials Restore Name:  example-acm-credentials-schedule-20220406171919
  Velero Resources Restore Name:    example-acm-resources-schedule-20220406171920
Events:
  Type    Reason                   Age   From                Message
  ----    ------                   ----  ----                -------
  Normal  Prepare to restore:      76m   Restore controller  Cleaning up resources for backup acm-credentials-hive-schedule-20220406155817
  Normal  Prepare to restore:      76m   Restore controller  Cleaning up resources for backup acm-credentials-cluster-schedule-20220406155817
```



