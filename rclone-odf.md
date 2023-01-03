# ODF와 Rclone 복제를 사용한 애플리케이션 백업 및 복구

이 예제에서는 Volsync 오퍼레이터와 OpenShift Data Foundation을 사용하여 클러스터 간 PV를 비동기식으로 복제하는 방법에 대해 설명합니다.

(퍼블릭 클라우드 상에 배포된 OpenShift 클러스터가 Volsync를 통해 PV 복제하는 방법에 대해서는 2022 Red Hat Container Day 웨비나가 진행되었으며, 또는 [이 문서](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.6/html-single/add-ons/index#volsync-install-clusters)를 참고하여 따라할 수 있습니다.)



## 개요

VolSync에서 제공하는 복제는 스토리지 시스템과 독립적이기 때문에, 일반적으로 원격 복제를 지원하지 않는 스토리지를 사용하는 경우에도 복제가 가능하며, 다양한 유형(및 공급업체)의 스토리지 간에도 복제할 수 있다는 장점이 있습니다. 



복제 방식은 크게 Rsync 기반 복제와 Rclone 기반 복제로 나눌 수 있습니다. 

Rsync 기반 복제는 ssh 연결을 통해 PV의 1:1 비동기 복제를 지원하며, 일반적으로 재해 복구 또는 원격 사이트로 데이터 전송을 위해 사용합니다.

Rclone 기반 복제는 AWS S3와 같은 중간 오브젝트 스토리지를 통해 하나의 영구 볼륨을 여러 위치에 복사합니다. 데이터를 여러 위치에 배포할 때 유용할 수 있습니다.



Rclone 기반 복제 방식에서 AWS S3와 같은 퍼블릭 클라우드의 오브젝트 스토리지를 사용하는 방법에 대해서는 잘 설명된 문서가 다수 있으므로 이 문서에서는 OpenShift Data Foundation의 object bucket을 통해 다른 클러스터로 PV를 복제하는 방법에 대해서 확인합니다.



## 전제 조건

- 2개 이상의 OpenShift 클러스터
- OpenShift `oc` CLI
- OpenShift Data Foundatin (ODF)의 subscription (OpenShift Container Platform Plus의 subscription 보유 시 포함되어있음)
- OpenShift 클러스터 중 하나는 ODF를 설치하기 위하여 worker node의 리소스의 합이 최소한 아래 조건을 만족하여야 합니다.
  - 24 CPU (logical)
  - 72 GiB memory
  - 3 storage devices
  - 예를 들어  8 CPU, 24 GiB memory, 1 storage device를 만족하는 워커노드를 3대 가지고 있는 클러스터는 위의 최소 조건을 만족할 수 있습니다.
- CSI 기반 스토리지 ( `oc get storageclasses`  명령어를 통해 `-csi` 로 끝나는 스토리지클래스 존재 여부 확인)



## 순서

### 1. ODF 스토리지 시스템 설치

전제 조건에서 언급한 ODF 설치 조건을 만족하는 클러스터를 **Cluster-1**, 나머지 클러스터를 **Cluster-2**라고 하겠습니다.

(테스트 용도로 최소한의 리소스를 사용하기 위하여 이 예제에서는 클러스터를 2개만 사용하였으나, rclone를 사용하는 사례에서는 원본 애플리케이션이 있는 클러스터(1)와 애플리케이션을 복구하는 클러스터(2) 외의 제 3의 클러스터에서 ODF를 구성하거나 다른 외부 오브젝트 스토리지를 사용하는 것이 좋습니다)

CLI 상에서 클러스터 간 전환의 편의를 위해 아래와 같이 context를 설정해줍니다. 

```
oc login [cluster-1의 주소]
oc config rename-context $(oc config current-context) c1
oc login [cluster-2의 주소]
oc config rename-context $(oc config current-context) c2
```



우선 콘솔을 통해 Cluster-1에 ODF 스토리지 시스템을 설치합니다.

![ODF-operator](img/volsync-operator.png)

OpenShift 콘솔 화면 (관리자 View) -> 왼쪽 탭의 Operator -> OperatorHub 에서 **OpenShift Data Foundation**을 검색하여 기본 설정값으로 설치합니다. (만약 아래쪽의 **콘솔 플러그인** 옵션이 비활성화로 선택되어있다면, 활성화 옵션을 선택해주세요)



몇 분 간 기다리면 설치가 완료되며, 웹콘솔을 새로고침해서 화면 왼쪽의 **스토리지** 아래에 **Data Foundation** 탭이 새롭게 추가된 것을 확인합니다. 

그리고 **Data Foundation** -> **스토리지 시스템** 탭에서 **스토리지 시스템 만들기 버튼**을 클릭합니다.

![image-20230103083104427](img/volsync-odf-new-1.png)



**배포 유형**은 'MultiCloud Object Gateway', **백업 저장소 유형**은 '기존 스토리지 클래스 사용' 선택, 스토리지 클래스는 '-csi'로 끝나는 스토리지 클래스를 선택합니다.

(예를 들어 vSphere 기반 클러스터는 thin-csi 선택)

![image-20230103090246697](img/volsync-odf-new-2.png)



용량은 0.5 TiB를 선택해도 괜찮으며, 노드는 3개 이상을 선택합니다. 이 외의 설정은 모두 기본값으로 두고 생성합니다.



스토리지 시스템이 생성 완료되기까지는 다소 오래 걸릴 수 있습니다. 아래 화면과 같이 상태가 Available로 표시될 때까지 기다립니다.

![image-20230103090723738](img/volsync-odf-new-3.png)



스토리지 시스템까지 모두 생성이 완료되면 왼쪽의 스토리지 -> Object Bucket Claims를 선택하여 `rclone-bucket` 이라는 이름을 Object Bucket Claim을 생성합니다.

![image-20230103090916134](img/volsync-odf-obc-1.png)



Object Bucket Claim이 생성되면 개체 버킷 클레임 세부 정보 페이지로 넘어가고, 아래쪽에서 다음과 같이 Endpoint 등의 정보를 확인할 수 있습니다.

여기서 **Bucket Name**을 따로 복사해둡니다.

![image-20230103091332284](img/volsync-odf-obc-2.png)







### 2. rclone 시크릿 생성

`oc` 커맨드를 통해 cluster-1에서 아래와 같은 명령어를 실행합니다.

```bash
oc config use-context c1
NOOBAA_ACCESS_KEY=$(oc get secret rclone-bucket -n openshift-storage -o json | jq -r '.data.AWS_ACCESS_KEY_ID|@base64d')
NOOBAA_SECRET_KEY=$(oc get secret rclone-bucket -n openshift-storage -o json | jq -r '.data.AWS_SECRET_ACCESS_KEY|@base64d')
ENDPOINT=$(oc get route s3 -n openshift-storage -o jsonpath='{.spec.host}')
```

```bash
cat << EOF >> rclone.conf
[ceph]
type = s3
provider = Ceph
env_auth = false
access_key_id = ${NOOBAA_ACCESS_KEY}
secret_access_key = ${NOOBAA_SECRET_KEY}
endpoint = http://${ENDPOINT}
region = 
location_constraint =
acl =
server_side_encryption =
storage_class =
EOF
```

```
oc new-project rocket-chat
oc create secret generic rclone-secret --from-file=rclone.conf -n rocket-chat
```



cluster-2로 전환하여 아래 명령어를 실행합니다.

```bash
oc config use-context c2
oc new-project rocket-chat
oc create secret generic rclone-secret --from-file=rclone.conf -n rocket-chat
```

(시크릿의 key는  `rclone.conf`가 되어야 함)







### 3. VolSync 오퍼레이터 설치

오퍼레이터 허브에서 VolSync 오퍼레이터를 지원하기 전에는 별도로 여러 구성요소를 설치해야 했으나, 이제는 오퍼레이터 허브를 통해 손쉽게 설치 가능합니다.

Operator -> OperatorHub에서 **VolSync** 오퍼레이터를 검색하여 기본 설정값으로 설치합니다.

cluster-1과 cluster-2에 모두 설치합니다.







### 4. cluster-1에 테스트용 앱 배포

cluster-1에서 아래 명령어를 실행합니다. 

```
oc config use-context c1
oc adm policy add-scc-to-user anyuid -z default -n rocket-chat
```



OpenShift 웹 콘솔 -> 스토리지 -> 스토리지 클래스 에서 csi 드라이버를 지원하는 스토리지 클래스가 기본으로 설정되어 있는지 확인합니다.

![image-20230103120324775](img/volsync-st-class-1.png)



**(1) 기본 스토리지 클래스를 변경하고 진행하는 경우**

기본으로 설정되어 있지 않다면, csi 지원 스토리지 클래스의 주석을 아래와 같이 추가하면 기본 스토리지 클래스로 변경할 수 있습니다. 

```
storageclass.kubernetes.io/is-default-class=true
```

그러고나서 애플리케이션을 배포합니다.

```
oc create -f https://raw.githubusercontent.com/kowillow/notes/main/.misc/rocket-chat.yaml
```



**(2) 기본 스토리지 클래스를 변경하지 않고 진행하는 경우**

기본 스토리지 클래스를 변경하고 싶지 않은 경우에는 예제 애플리케이션 용으로 PVC를 생성해주어야 합니다. 

PVC 이름을 `rocketchat-data-claim` 으로 설정, 스토리지 클래스는 csi 드라이버 지원 스토리지 클래스(클러스터의 환경에 따라 다름)로 설정, 크기는 10GiB로 설정하여 `rocket-chat` 프로젝트에 생성합니다.

![image-20230103120648377](img/volsync-st-class-2.png)



 `rocketchat-data-claim` PVC 생성 후 아래 명령어를 사용하여 예제 애플리케이션을 배포합니다.

```
oc create -f https://raw.githubusercontent.com/kowillow/notes/main/.misc/rocket-chat-without-pv.yaml
```

> DB의 deploy와 post hook 파드가 모두 성공적으로 완료되어야 APP 파드가 정상적으로 실행 될 수 있습니다. 따라서 그 전까지는 rocketchat 파드가 Error나 CrushLoopBackOff를 반복하며 몇 번 재시작될 수 있습니다. 



DB 파드가 완료 및 running 되고 rocketchat 파드 또한 running 되면 아래 명령어를 통해 애플리케이션의 주소를 확인합니다. 또는 콘솔에서 네트워킹-> 경로를 통해 rocket-chat의 주소를 확인합니다. 

```
oc get route rocket-chat -n openshift-storage -n rocket-chat
```



해당 주소로 접속하면 첫 접속이므로 설치 정보를 입력하게 됩니다. 관리자 정보를 편한대로 입력하고, 4단계 서버 등록에서 '독립 실행형'을 선택하여 설치를 완료합니다.

![image-20230103102741285](img/volsync-rocket-install.png)



서버 등록을 완료하고 채팅방으로 접속하면 아래와 같은 화면을 확인할 수 있습니다. 기본적으로 생성되어있는 general 채팅방으로 이동하여 임의의 메시지를 입력합니다.

![image-20230103102958985](img/volsync-rocket-channel.png)







### 5. Replication CRD 생성

이제 Volsync를 통해 cluster-1의 PV를 cluster-2로 주기적으로 복제해주는 CRD를 생성합니다.

- **Cluster-1**에서 ReplicationSource 생성

```yaml
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: rocket-source
  namespace: rocket-chat
spec:
  rclone:
    copyMethod: Snapshot
    rcloneConfig: rclone-secret
    rcloneConfigSection: ceph
    rcloneDestPath: ## rclone-buekct의 버킷 이름
  sourcePVC: rocketchat-data-claim
  trigger:
    schedule: '*/2 * * * *'
```



- **Cluster-2**에서 ReplicationDestination 생성

```yaml
apiVersion: volsync.backube/v1alpha1
kind: ReplicationDestination
metadata:
  name: rocket-dest
  namespace: rocket-chat
spec:
  trigger:
    schedule: '*/2 * * * *'
  rclone:
    copyMethod: Snapshot
    rcloneConfig: rclone-secret
    rcloneConfigSection: ceph
    rcloneDestPath: ## rclone-buekct의 버킷 이름
    accessModes: [ReadWriteOnce]
    capacity: 10Gi
```



CRD가 생성되고 Volsync가 정상적으로 복제를 완료하면 cluster-2에서 아래와 같이 복제된 스냅샷을 확인할 수 있습니다.

![image-20230103122613007](img/volsync-snapshot-1.png)



### 6. cluster-2에서 애플리케이션 복구

아래와 같은 명령어를 실행합니다. 

```
oc config use-context c2
oc adm policy add-scc-to-user anyuid -z default -n rocket-chat
```



cluster-2의 웹콘솔의 스토리지 -> 볼륨 스냅샷 -> 스냅샷 오른쪽의 `...` 를 클릭 -> 새 PVC로 복원를 클릭합니다.

이름은 ` rocketchat-data-claim`, 스토리지 클래스는 csi 호환 스토리지 클래스를 선택하여 아래쪽의 '복원' 버튼을 클릭합니다.

![image-20230103123453117](img/volsync-snapshot-2.png)



PVC는 바로 바인딩 되지 않고 consumer를 기다리면서 pending 됩니다. 이어서 애플리케이션을 배포합니다.

```
oc create -f https://raw.githubusercontent.com/kowillow/notes/main/.misc/rocket-chat-without-pv.yaml
```



파드가 모두 running 되고, route 주소를 통해 접근하면 처음 cluster-1에서 '설치 마법사' 화면이 나왔던 것과 다르게 아래와 같이 로그인 화면이 표시됩니다. cluster-1에서 생성했던 사용자명과 비밀번호로 로그인 합니다.

![image-20230103120149766](img/volsync-rocket-login.png)



그러면 아래와 같이 cluster-1의 rocket chat에서 입력했던 메시지를 cluster-2에서 복원한 rocket chat에서도 확인할 수 있습니다.

![image-20230103102958985](img/volsync-rocket-channel.png)





## 리소스 삭제

테스트가 끝났으면 아래 명령어를 통해 클러스터에서 리소스를 삭제합니다.

```
oc delete -f https://raw.githubusercontent.com/kowillow/mig-demo-apps/main/manifest.yaml
oc delete project rocket-chat
```

