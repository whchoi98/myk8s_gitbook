---
description: 'Update : 2023-05-06'
---

# Stateful Container-EBS

### EBS CSI 구성확인

[**볼륨/CSI**](stateful-container.md#csi) 에서 생성된 각 노드별 CSI DaemonSet, CSI Contorller 가 생성되어 있는지 확인해 봅니다.

```
kubectl -n kube-system get pods | grep csi
kubectl -n kube-system get pods ebs-csi-controller-6bd865f7dd-68x72 -o jsonpath={..spec.containers[*].name} | xargs -n1

```

CSI 내부에는 6개의 Container가 동작하고 있습니다.

```
ebs-plugin csi-provisioner csi-attacher csi-snapshotter csi-resizer liveness-probe

```

## 스토리지 클래스 생성.&#x20;

동적 볼륨 프로비저닝을 사용하면 필요에 따라 스토리지 볼륨을 생성할 수 있습니다. StorageClass는 동적 프로비저닝이 호출될 때 사용해야 하는 프로비저너(csi-provisioner)에 전달해야 하는 매개변수를 정의하기 위해 미리 생성되어야 합니다.

먼저 클러스터가 사용할 스토리지 클래스를 정의해야 하고, PV 클레임에 대한 기본 스토리지 클래스를 정의해야 합니다.

스토리지 클래스를 아래와 같이 구성해 두었습니다.

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: mysql-gp2
provisioner: ebs.csi.aws.com # Amazon EBS CSI driver
parameters:
  type: gp2
  encrypted: 'true' 
volumeBindingMode: WaitForFirstConsumer 
reclaimPolicy: Delete
mountOptions:
- debug
```

* provisioner - `ebs.csi.aws.com`.
* volume type - GP2
* encrypted - EBS 볼륨 암호화
* volumeBindingMode - WaitForFirstConsumer (Pod가 생성된 후 동일한 AZ에 상주하도록 persistent Volume이 프로비저닝되도록 하기 위해 volumeBindingMode는 WaitForFirstConsumer로 구성)

Storage Class를 생성합니다.

```
kubectl apply -f ${HOME}/environment/myeks/ebs_statefulset/mysql-storageclass.yaml

```

생성된 스토리지 클래스를 확인합니다.

```
kubectl describe storageclass mysql-gp2

```

출력 결과를 확인합니다.

```
whchoi:~/environment/ebs_statefulset $ kubectl describe storageclass mysql-gp2
Name:            mysql-gp2
IsDefaultClass:  No
Annotations:     kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{},"name":"mysql-gp2"},"mountOptions":["debug"],"parameters":{"encrypted":"true","type":"gp2"},"provisioner":"ebs.csi.aws.com","reclaimPolicy":"Delete","volumeBindingMode":"WaitForFirstConsumer"}

Provisioner:           ebs.csi.aws.com
Parameters:            encrypted=true,type=gp2
AllowVolumeExpansion:  <unset>
MountOptions:
  debug
ReclaimPolicy:      Delete
VolumeBindingMode:  WaitForFirstConsumer
Events:             <none>
```

mysql-gp2가 storageClassName으로 선언되었습니다.

## ConfigMap 생성

컨피그맵은 key-vlaue Pair로 기밀이 아닌 데이터를 저장하는 데 사용하는 API 오브젝트입니다. [파드](https://kubernetes.io/ko/docs/concepts/workloads/pods/pod-overview/)는 [볼륨](https://kubernetes.io/ko/docs/concepts/storage/volumes/) 내에서 환경 변수, 커맨드-라인 인수 또는 구성 파일로 컨피그맵을 사용할 수 있습니다. 이러한 컨피그맵을 사용하면 [컨테이너 이미지](https://kubernetes.io/ko/docs/reference/glossary/?all=true#term-image)에서 환경별 구성을 분리하여, 애플리케이션을 쉽게 이식할 수 있습니다.

### 1.namespace /configmap 생성

Stateful DB를 생성할 것입니다. 새로운 namespace를 생성합니다.

```
kubectl create namespace mysql

```

Configmap을 생성합니다.

```
kubectl apply -f ~/environment/myeks/ebs_statefulset/mysql-configmap.yaml 

```

configmap은 아래와 같이 구성되어 있습니다.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: mysql
  labels:
    app: mysql
data:
  master.cnf: |
    # Apply this config only on the leader.
    [mysqld]
    log-bin
  slave.cnf: |
    # Apply this config only on followers.
    [mysqld]
    super-read-only
```

ConfigMap은 master.cnf, slave.cnf를 저장하고 StatefulSet에 정의된 leader 및 follower pod를 초기화할 때 전달합니다. master.cnf는 데이터 변경 기록을 제공하기 위해 바이너리 로그 옵션(log-bin)이 있는 MySQL leader pod용입니다. 팔로어 서버로 전송됩니다. slave.cnf는 super-read-only 옵션이 있는 follower 포드용입니다.

## Service 생성

Kubernetes Service는 Pod의 논리적 집합들과 액세스하는 정책을 정의합니다. serviceSpec에서 유형을 지정하고, 다양한 방식으로 서비스를 노출할 수 있습니다. StatefulSet는 현재 포드의 도메인을 제어하고 안정적인 DNS 항목으로 각 포드에 직접 도달하기 위해 헤드리스 서비스가 필요합니다. clusterIP에 "None"을 지정하면 헤드리스 서비스를 생성할 수 있습니다.

이것은 Amazon Aurora 에서 Endpoint 를 기반으로 Cluster를 구성하는 것과 유사합니다.

아래와 같이 Service를 Apply 합니다.

```
kubectl apply -f ${HOME}/environment/myeks/ebs_statefulset/mysql-services.yaml

```

실제 yaml을 확인해 봅니다. (mysql-services.yaml)

```
# Headless service for stable DNS entries of StatefulSet members.
apiVersion: v1
kind: Service
metadata:
  namespace: mysql
  name: mysql
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  clusterIP: None
  selector:
    app: mysql
---
# Client service for connecting to any MySQL instance for reads.
# For writes, you must instead connect to the leader: mysql-0.mysql.
apiVersion: v1
kind: Service
metadata:
  namespace: mysql
  name: mysql-read
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  selector:
    app: mysql
EoF

```

mysql service는 DNS 확인용이므로 StatefulSet 컨트롤러에 의해 포드가 배치될 때 pod-name.mysql을 사용하여 포드를 확인할 수 있습니다. mysql-read는 모든 follower에 대한 로드 밸런싱을 수행하는 클라이언트 서비스입니다.

## StatefuleSet 구성

mysql-statefulset.yml을 기반으로 StatefuleSet을 구성합니다.

```
kubectl apply -f ${HOME}/environment/myeks/ebs_statefulset/mysql-statefulset.yaml

```

정상적으로 생성되었는지 확인합니다.&#x20;

{% hint style="success" %}
생성 되는데 5분 정도 시간이 소요됩니다.
{% endhint %}

```
kubectl -n mysql rollout status statefulset mysql

```

동적 생성된 PVC를 확인합니다.

```
kubectl -n mysql get pvc -l app=mysql

```

아래와 같은 결과를 확인할 수 있습니다.

```
whchoi:~/environment/ebs_statefulset $ kubectl -n mysql get pvc -l app=mysql
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-mysql-0   Bound    pvc-82184a36-d4c4-4371-8deb-a426836af336   10Gi       RWO            mysql-gp2      2m41s
data-mysql-1   Bound    pvc-b492e2aa-1539-447f-a79e-f2316a80dea3   10Gi       RWO            mysql-gp2      116s
```

PVC를 아래 명령을 통해 확인합니다.

```
kubectl -n mysql get pvc -l app=mysql

```

아래와 같은 출력 결과를 확인 할 수 있습니다.

```
whchoi98:~/environment/myeks (master) $ kubectl -n mysql get pvc -l app=mysql
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-mysql-0   Bound    pvc-c08c7ae2-e3da-46a8-8910-54ac87e0ee20   10Gi       RWO            mysql-gp2      4m58s
data-mysql-1   Bound    pvc-f68bfe62-2ce3-4371-b89a-def75716a106   10Gi       RWO            mysql-gp2      4m11s
            mysql-gp2      3m16s
```

EC2 대시보드의 볼륨에서도 동일한 결과를 확인 할 수 있습니다.

![](<../.gitbook/assets/image (381).png>)

## SQL Test&#x20;

### 1.SQL 테스팅.

다음 명령을 실행하여 **mysql-client** 를 사용해서 일부 데이터를 **mysql-0.mysql** 에 보낼 수 있습니다 .

```
kubectl -n mysql run mysql-client --image=mysql:5.7 -i --rm --restart=Never --\
  mysql -h mysql-0.mysql <<EOF
CREATE DATABASE test;
CREATE TABLE test.messages (message VARCHAR(250));
INSERT INTO test.messages VALUES ('hello, from mysql-client');
EOF

```

정상적으로 read되는 지 확인해 봅니다.

```
kubectl -n mysql run mysql-client --image=mysql:5.7 -it --rm --restart=Never --\
>   mysql -h mysql-read -e "SELECT * FROM test.messages"

```

아래와 같이 출력됩니다.

```
whchoi:~/environment/ebs_statefulset $ kubectl -n mysql run mysql-client --image=mysql:5.7 -it --rm --restart=Never --\
>   mysql -h mysql-read -e "SELECT * FROM test.messages"
+--------------------------+
| message                  |
+--------------------------+
| hello, from mysql-client |
+--------------------------+
```

상호간 분산 테스트를 위해서 아래 명령을 통해 확인 해 봅니다.

```
kubectl -n mysql run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
   bash -ic "while sleep 1; do mysql -h mysql-read -e 'SELECT @@server_id,NOW()'; done"

```

### 2. Container 장애 시험

새로운 Cloud9창에서 아래 명령을 통해 MySQL을 응답하지 않는 상태로 만듭니다.

```
kubectl -n mysql exec mysql-1 -c mysql -- mv /usr/bin/mysql /usr/bin/mysql.off

```

앞서 상호 분산테스트 하였던 Cloud9창에서 어떤 변화가 일어나는지 확인해 봅니다. 아래 명령을 통해 mysql 상태를 확인해 봅니다.

```
kubectl -n mysql get pod mysql-1

```

아래와 같은 출력결과를 확인할 수 있습니다.

```
whchoi:~/environment $ kubectl -n mysql get pod mysql-1
NAME      READY   STATUS    RESTARTS   AGE
mysql-1   1/2     Running   0          15m
```

다시 정상으로 복귀 시키고, 확인해 봅니다.

```
kubectl -n mysql exec mysql-1 -c mysql -- mv /usr/bin/mysql.off /usr/bin/mysql
kubectl -n mysql get pod mysql-1

```

아래와 같은 결과를 확인할 수 있습니다.

```
whchoi:~/environment $ kubectl -n mysql get pod mysql-1
NAME      READY   STATUS    RESTARTS   AGE
mysql-1   2/2     Running   0          17m
```

### 3.Pod 삭제 시험

실패한 포드를 시뮬레이션하려면 다음 명령을 사용하여 mysql-1 포드를 삭제합니다.

```
kubectl -n mysql delete pod mysql-1

```

앞서 Cloud9 창에서 다시 100에서만 응답되는 것을 확인할 수 있습니다.

StatefulSet 컨트롤러는 실패한 포드를 인식하고 동일한 이름과 링크를 가진 복제본 수를 유지하기 위해 새 포드를 만듭니다.

```
kubectl -n mysql get pod mysql-1 -w

```

```
whchoi:~/environment $ kubectl -n mysql get pod mysql-1 -w
NAME      READY   STATUS    RESTARTS   AGE
mysql-1   2/2     Running   0          40s
```

### 4.Scaling 시험

이제 스케일링을 시험해 봅니다. replica를 3개까지 확장합니다.

```
kubectl -n mysql scale statefulset mysql --replicas=3

```

아래 명령을 통해 확장되는 것을 확인합니다.

```
kubectl -n mysql rollout status statefulset mysql

```

다시 앞서 Cloud9 창에서 다시 100,101,102에서 분산해 Read가 일어나는 것을 확인 할 수 있습니다.

새로 배포 된 팔로워 (mysql-2)가 다음 명령으로 동일한 데이터 세트를 가지고 있는지 확인합니다. 동일한 메세지가 출력됩니다.

```
kubectl -n mysql run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
 mysql -h mysql-2.mysql -e "SELECT * FROM test.messages"
 
```

이제 다시 축소해 봅니다. 하지만 데이터 또는 PVC를 삭제하지는 않습니다. 수동삭제 해야 합니다.

```
kubectl -n mysql  scale statefulset mysql --replicas=2
kubectl -n mysql get pods -l app=mysql

```

여전히 PVC가 존재하는 지 확인해 봅니다.

```
kubectl -n mysql  get pvc -l app=mysql

```

```
whchoi:~/environment $ kubectl -n mysql  get pvc -l app=mysql
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-mysql-0   Bound    pvc-82184a36-d4c4-4371-8deb-a426836af336   10Gi       RWO            mysql-gp2      31m
data-mysql-1   Bound    pvc-b492e2aa-1539-447f-a79e-f2316a80dea3   10Gi       RWO            mysql-gp2      31m
data-mysql-2   Bound    pvc-ba84a117-2dc0-455b-ae2a-206466191114   10Gi       RWO            mysql-gp2      8m35s
```

기본적으로 PersistentVolumeClaim을 삭제하면 연결된 영구 볼륨이 삭제됩니다. 볼륨을 유지하고 싶다면, "data-mysql-2"라는 PersistentVolumeClaim과 연결된 PersistentVolume의 reclaim 정책을 "Retain"으로 변경합니다.

```
export pv=$(kubectl -n mysql  get pvc data-mysql-2 -o json | jq --raw-output '.spec.volumeName')
echo data-mysql-2 PersistentVolume name: ${pv}
kubectl -n mysql patch pv ${pv} -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
kubectl get persistentvolume

```

이제 PersistentVolumeClaim data-mysql-2를 삭제해도 AWS EC2 콘솔에서 상태가 "사용 가능" 상태인 EBS 볼륨을 계속 볼 수 있습니다.&#x20;

```
$ kubectl get persistentvolume
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   REASON   AGE
pvc-2817c371-ad84-4ebf-a44f-0bec4b269c99   10Gi       RWO            Delete           Bound    ebs-pv-test/ebs-pvc-01   ebs-sc-01               45m
pvc-79a9f9c3-7fb3-4956-b2ea-786f00a47244   10Gi       RWO            Retain           Bound    mysql/data-mysql-2       mysql-gp2               7m42s
pvc-db4bb2a0-b053-4a4b-95ce-9282eb005489   10Gi       RWO            Delete           Bound    mysql/data-mysql-1       mysql-gp2               15m
pvc-e8864866-abe0-451c-bf3e-f72b6b53ea9e   10Gi       RWO            Delete           Bound    mysql/data-mysql-0       mysql-gp2               16m
```

reclaim 정책을 다시 "Delete"로 변경합니다.

```
kubectl patch pv ${pv} -p '{"spec":{"persistentVolumeReclaimPolicy":"Delete"}}'
unset pv
kubectl get persistentvolume
```

아래와 같이 PVC를 삭제하면 PV는 삭제 됩니다. mtsql-2 pvc를 삭제합니다.

```
kubectl -n mysql delete pvc data-mysql-2
kubectl get persistentvolume

```
