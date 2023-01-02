# Red Hat OpenShift와 Red Hat Streams for Apache Kafka를 사용한 이벤트 기반 아키텍처 확장

Apache Kafka 및 OpenShift를 사용하는 음식 배달 예제 애플리케이션을 배포해보도록 하겠습니다. 

(예제 애플리케이션 소스코드 및 설명: https://github.com/IBM/scaling-apps-with-kafka)



예제 애플리케이션은 Kafka topic을 활용하여 주문 레코드를 생성하고 소비합니다. 그리고 CPU 및 메모리 사용량에 따라 확장하는 HPA(Horizontal Pod Autoscaler) 대신, 수신 메시지를 기반으로 크기를 조정하는 Custom Metric Autoscaler 오퍼레이터(KEDA 기반 오퍼레이터)를 함께 사용하겠습니다.



가장 최근에 Producer가 생성한 메시지와 Consumer가 소비한 현재 메시지 간의 차이가 커지기 시작하면 이는 보통 Consumer가 Kafka topic에 들어오는 레코드 또는 메시지를 따라갈 수 없음을 의미합니다. Custom Metric Autoscaler를 사용하면 이를 감지하여 더 많은 메시지를 소비하고 처리하여 들어오는 메시지의 속도를 따라잡을 수 있도록 Consumer 수를 자동으로 조절할 수 있습니다.



![건축학](img/architecture.png)



## 전제 조건

- OpenShift 클러스터
- OpenShift CLI(oc)



## 순서

1. Repository 복제
2. dd





### 1. OpenShift 클러스터

먼저 예제 애플리케이션을 배포하기 위해 오픈시프트 클러스터에 `food-delivery` 프로젝트를 새로 생성합니다. 아래와 같이 CLI를 이용하거나, 콘솔을 사용할 수도 있습니다.

```bash
oc new-project food-delivery
```



이어서 관리자 view -> Operator -> OperatorHub에서 `Custom Metrics Autoscaler`를 기본값 설정으로 설치합니다.

해당 오퍼레이터는 `openshift-keda` 프로젝트에 설치되며, 설치가 완료되기까지 몇 분의 시간이 소요될 수 있습니다.

![image-20230102161921690](img/operatorhub.png)



오퍼레이터 설치가 완료되면 `KedaController` 인스턴스 생성 선택 -> 기본값(이름: keda)으로 인스턴스를 생성합니다.

![image-20230102162326819](img/operator-kedacontroller.png)



잠시 기다려 KedaController 인스턴스의 상태가 Phase: Installation Succeeded 임을 확인합니다.



### 2. REPO 복제

그리고 예제 애플리케이션의 소스가 있는 `scaling-apps-with-kafka` 리포지토리를 로컬로 복제합니다. 

```bash
git clone https://github.com/IBM/scaling-apps-with-kafka
```



### 3. Kafka 서비스 생성

레드햇 아이디가 있으면, [레드햇 애플리케이션 서비스 웹사이트](https://console.redhat.com/application-services/streams/kafkas)에서 48시간 동안 동작하는 무료 kafka instance를 생성할 수 있습니다. 

![image-20230102153039691](img/kafka-webservice.png)

이름 및 각종 설정값은 임의로 작성 및 선택하여 생성하면 됩니다. 잠시 기다리면 kafka 인스턴스가 성공적으로 생성되었음을 확인할 수 있습니다.



그 후, 생성한 kafka 인스턴스를 클릭하고 Topics 탭에서 새로운 topic을 생성합니다.

아래와 같이 이름은 `orders`, partitions의 수는 6으로 설정, 나머지는 기본값으로 두고 새로운 topic를 생성합니다.

![image-20230102154649608](img/kafka-topic.png)



그리고 맨 오른쪽의 `...` 아이콘 -> `connection`을 선택합니다.

![image-20230102154810576](img/kafka-connection-1.png)





![image-20230102155741504](img/kafka-connection-2.png)

그러면 위와 같은 화면이 보입니다. 여기서 확인할 수 있는 Bootstrap Server 주소를 복사해두합니다.

그리고 `Create service account` 버튼을 클릭하고, 본인의 account를 알아볼 임의의 설명을 입력하면 Client ID와 Client Secret 값을 얻을 수 있습니다. 이 값 또한 따로 복사해둡니다.



이어서 Access 탭에서 Manage Access 버튼을 클릭하여 권한을 수정합니다. 

1. 먼저 대상 account는 All account 또는 방금 생성한 service account를 선택합니다. 

2. 그리고 Assign permission 부분에서 Add permission -> 'Consume from topic'와 'Produce to a topic'를 각각 클릭하면 아래와 같은 필드들이 추가됩니다. 
3. 대상 topic과 consumer group값은 아래와 같이 **IS '*'** 로 설정하고, Save를 눌러 권한 수정 내용을 저장합니다.

![image-20230102163206679](img/kafka-permissions.png)



 ### 4. 애플리케이션 배포

애플리케이션을 배포하기에 앞서, kafka 인스턴스와의 통신에 사용할 인증 정보를 입력합니다.

 `deployments/kafka-secret.yaml` 파일에서 BOOTSRAP_SERVERS 값을 복사해둔 **Bootstrap Server** 주소로 바꿉니다.

그리고 SASL_USERNAME 및 SASL_PASSWORD에는 각각 **Client ID**와 **Client Secret**값을 대신 넣어줍니다.

```
...
  BOOTSTRAP_SERVERS: 'kafka:9092'
  SECURITY_PROTOCOL: 'SASL_SSL'
  SASL_MECHANISMS: 'PLAIN'
  SASL_USERNAME: '****'
  SASL_PASSWORD: '***'
...
```



각 애플리케이션 서비스가 Kafka 서비스에 접근하는 데에 사용할 수 있도록 OpenShift 클러스터에 시크릿을 생성합니다.

```
oc apply -f deployments/kafka-secret.yaml
```



그리고 MongoDB 및 Redis 인스턴스를 먼저 배포합니다. 

```
oc apply -f deployments/mongo-dev.yaml
oc apply -f deployments/redis-dev.yaml
```



그런 다음 애플리케이션 서비스를 배포합니다.

```
oc apply -f deployments/frontend.yaml
oc apply -f deployments/apiservice.yaml
oc apply -f deployments/statusservice.yaml
oc apply -f deployments/orderconsumer.yaml
oc apply -f deployments/kitchenconsumer.yaml
oc apply -f deployments/courierconsumer.yaml
oc apply -f deployments/realtimedata.yaml
oc apply -f deployments/podconsumerdata.yaml
```



### 5. KEDA ScaledObjects 배포

ScaledObjects를 배포하기 전에 Custom Metric Autoscaler (KEDA) 또한 Kafka 인스턴스에 접근할 수 있도록 `deployments/keda-auth.yaml` 파일을 사용하여  `TriggerAuthentication`를 만듭니다. 이 `TriggerAuthentication`은 앞 단계에서 생성한 `kafka-credentials` 시크릿을 사용하여 인증합니다.

```
oc apply -f deployments/keda-auth.yaml
```



이제 `TriggerAuthentication`을 참조하는  `ScaledObject`를 만들 수 있습니다.

우선 `deployments/keda-scaler.yaml` 파일에서 `bootstrapServers` 값을 아까 복사한 **Bootstrap Server** 주소로 수정합니다. 

> 부트스트랩 서버 주소는 아까 수정한 deployments/kafka-secret.yaml 파일 또는 kafka 서비스 웹사이트에서 확인할 수 있습니다.



아래와 같은 줄을 모두 수정합니다.

```
          bootstrapServers: pkc-lzvrd.us-west4.gcp.confluent.cloud:9092
```



`ScaledObject`들의 lagThreshold 값을 확인하면 모두 5로 설정되어있습니다. 이 값은 오토스케일링을 실행하는 기준이 되는 지연 대상 값입니다. 현재 Consumer group에서 스트림이 얼마나 지연되고 있는지 표시하는 값이며, 5가 되면 스트림 처리 속도가 지연되고 있는 것으로 판단하여 오토스케일링을 시작합니다.

아래 명령어로 `ScaledObject`들도 생성해줍니다. 

```
oc apply -f deployments/keda-scaler.yaml
```





### 6. 애플리케이션 실행

애플리케이션에 접근하려면 프런트엔드와 마이크로서비스를 노출해야 합니다.

`oc` CLI를 사용하여 프런트엔드를 노출합니다 .

```
oc expose svc/example-food
```



그리고 백엔드 서비스를 노출하기 전에 프런트엔드의 호스트 이름을 가져와서 동일한 호스트 이름을 백엔드에 사용하도록 합니다. 

```bash
oc get route example-food -o jsonpath='{.spec.host}'
example-food-food-delivery.***.appdomain.cloud
```



`deployments/routes.yaml` 파일에서 `host: HOSTNAME` 부분의 **HOSTNAME**을 아래와 같이 교체합니다. 

```
host: example-food-food-delivery.***.appdomain.cloud
```

그리고 APi version도 ` v1`에서  `route.openshift.io/v1`로 변경합니다.

```
apiVersion: route.openshift.io/v1
```



그런 다음 yaml 파일을 적용하여 route를 배포합니다.

```
oc apply -f deployments/routes.yaml
```

route는 URL 경로를 의도한 대상 service에 매핑합니다. `oc get routes` 명령어를 사용하면 생성된 route들을 확인할 수 있습니다.



이제 터미널을 사용하여 API를 실행할 수 있습니다. 명령에서 **YOUR_HOSTNAME**을 본인의 호스트 이름으로 바꿔서 실행합니다. 프런트엔드의 모바일 시뮬레이터에 레스토랑 목록이 채워지면서 아래와 비슷한 응답을 받아야 합니다. 

```
curl -X POST -H "Content-Type: application/json" -d @restaurants.json http://YOUR_HOSTNAME/restaurants

### OUTPUT
{"requestId":"e2bdbb26-b8ba-406a-acbe-b10e2cfd599b","payloadSent":{"restaurants":[...]}}
```





## 테스트

호스트 이름으로 접속하면 브라우저에서 시뮬레이터를 확인할 수 있습니다. (예: example-food-food-delivery.***.appdomain.cloud). 



하단 섹션의 슬라이더를 조정하여 주문 속도와 주방 및 배달의 처리 속도를 조정할 수 있습니다. 마이크로서비스(Status, Order, Driver, Kitchen Service) 위의 숫자는 파드의 수이고, 아래 숫자는 사용 중인 메시지의 수입니다.

![img](img/sample-output-1.png)

먼저 배달원(couriers)과 주방(kitchen)의 속도는 최대, 주문(order) 속도는 최소로 설정해서 START를 누르면 오른쪽에 생성된 주문과 완료된 주문의 그래프 데이터 포인트가 표시되기 시작합니다. Consumer가 초당 1개의 주문을 처리할 수 있으므로 파드(Pod)의 수는 동일하게 유지됩니다.



![img](img/sample-output-2.png)

그런 다음 주문 속도를 최대로 높이면, 주문을 처리하기 위해 Driver와 Kitchen 파드 수가 증가하는 것을 확인할 수 있습니다. 

`scaeldobject` 설정에서 최소 레플리카 수를 2, 최대 레플리카 수를 6으로 설정했기 때문에 파드 수는 최대 6개까지만 증가합니다.
