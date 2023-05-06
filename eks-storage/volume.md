# 볼륨/CSI

## 소개



컨테이너 내의 디스크에 있는 파일은 임시적이며, 컨테이너에서 실행될 때 애플리케이션에 적지 않은 몇 가지 문제가 발생합니다. 한 가지 문제는 컨테이너가 크래시될 때 파일이 손실된다는 것입니다. kubelet은 컨테이너를 다시 시작하지만 초기화된 상태입니다. 두 번째 문제는 `Pod`에서 같이 실행되는 컨테이너간에 파일을 공유할 때 발생합니다. 쿠버네티스 [볼륨](https://kubernetes.io/ko/docs/concepts/storage/volumes/) 추상화는 이러한 문제를 모두 해결합니다.



## 1.임시볼륨 (empty volume)

파드가 시작될 때 빈 상태로 시작되며, 저장소는 로컬의 kubelet 베이스 디렉터리(보통 루트 디스크) 또는 램에서 제공됩니다.

아래와 같이 새로운 임시볼륨을 생성해 봅니다.

```
cat << EOF > ~/environment/empty_stg.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: empty
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-volume-1
  namespace: empty
spec:
  containers:
  - name: container1
    image: whchoi98/network-multitool
    volumeMounts:
    - name: empty-dir
      mountPath: /mount1
  - name: container2
    image: kubetm/init
    volumeMounts:
    - name: empty-dir
      mountPath: /mount2
  volumes:
  - name : empty-dir
    emptyDir: {}
EOF

```

생성된 yml 파일을 아래 명령어로 실행합니다.

```
kubectl apply -f  ~/environment/empty_stg.yaml
kubectl -n empty get pods

```

생성된 첫번째 컨테이너 (Container 1)의 Shell에 접속해서 아래 명령을 수행합니다. \
(앞서 설치한 K9를 활용해서 접속할 수 있습니다.)

<figure><img src="../.gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

```
echo "This is empty-volumes" >> /mount1/empty.txt
cat /mount1/empty.txt

```

생성된 두번째 컨테이너 (Container2)의 Shell에 접속해서 Container1 에서 생성한 텍스트 값이 있는지 확인해 봅니다.

<figure><img src="../.gitbook/assets/image (11).png" alt=""><figcaption></figcaption></figure>

```
cat /mount2/empty.txt

```

임시 볼륨은 Pod내에서만 존재하기 때문에 Pod가 삭제되면 더 이상 데이터는 보존되지 않습니다.



## 2.hostPath

`hostPath` 볼륨은 호스트 노드의 파일시스템에 있는 파일이나 디렉터리를 파드에 마운트하는 방식입니다. hostPath는 호스트 내의 Pod들이 볼륨을 공유하는 방식이기 때문에 Worker Node가 삭제되면 데이터도 삭제됩니다.

이미 생성된 Node들 중에서 1개의 노드를 선택해서 새로운 라벨링을 부여합니다.

아래는 구성 예제입니다.

```
kubectl get nodes -l nodegroup-type=managed-frontend-workloads
kubectl label nodes {host-path-nodes} hostpath=true
kubectl get nodes -l hostpath=true

```

새로운 네임스페이스와 Pod 1개를 구성합니다.

```
cat << EOF > ~/environment/hostpath_stg1.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: hostpath
---
apiVersion: v1
kind: Pod
metadata:
  name: hostpath1
  namespace: hostpath
spec:
  containers:
  - name: container1
    image: whchoi98/network-multitool
    volumeMounts:
    - name: hostpath
      mountPath: /hostpath-mount
  volumes:
  - name: hostpath
    hostPath:
      path: /tmp
      type: Directory
  nodeSelector:
    hostpath: "true"
EOF
kubectl apply -f  ~/environment/hostpath_stg1.yaml
kubectl -n hostpath get pods -o wide

```

&#x20;container1에 shell로 접근해서 새로운 Text 파일을 만들어 봅니다. 아래와 같이 k9s를 통해서 Container1에 Shell로 접속합니다.

<figure><img src="../.gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>

```
echo "This is hostpath-mount-01" >> /hostpath-mount/hostpath.txt

```

하나의 pod를 추가로 생성해 봅니다.

```
cat << EOF > ~/environment/hostpath_stg2.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: hostpath
---
apiVersion: v1
kind: Pod
metadata:
  name: hostpath2
  namespace: hostpath
spec:
  containers:
  - name: container2
    image: whchoi98/network-multitool
    volumeMounts:
    - name: hostpath
      mountPath: /hostpath-mount
  volumes:
  - name: hostpath
    hostPath:
      path: /tmp
      type: Directory
  nodeSelector:
    hostpath: "true"
EOF
kubectl apply -f  ~/environment/hostpath_stg2.yaml
kubectl -n hostpath get pods -o wide


```

새롭게 생성된 Container2에 Shell로 접근해서 아래 명령을 수행해 봅니다.

```
echo "This is hostpath-mount-02" >> /hostpath-mount/hostpath.txt

```

이제 앞서 **`hostpath=true`** 라벨링을 부여한 노드로 접속해서, 실제 노드 값을 확인해 봅니다.

Session Manager를 통해서 접속이 가능합니다.

<figure><img src="../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (1) (5).png" alt=""><figcaption></figcaption></figure>

아래와 같은 방법으로도 확인이 가능합니다.

```
 # EC2 IP와 instance id 조회
 ~/environment/useful-shell/aws_ec2_ext.sh

```

나온 결과의 실제 EC2 Node IP를 확인하고, Session Manager로 접속해 봅니다.

```
 # host-path로 Labeling되어 있는 EC2의 instanace id로 접근
 aws ssm start-session --target {instance-id}
```

아래와 같이 EC2 콘솔에서 값을 조회해 봅니다.

```
cat /tmp/hostpath.txt

```

노드에서 실행한 값이 앞서 Container1,2에서 실행한 값을 모두 포함하는지 확인해 봅니다.

노드기반의 hostPath는 노드가 삭제되면 모든 데이터는 소멸됩니다.



## 3. 퍼시스턴트 볼륨

퍼시스턴트볼륨은 사용자 및 관리자에게 스토리지 사용 방법에서부터 스토리지가 제공되는 방법에 대한 세부 사항을 추상화하는 API를 제공합니다. 이를 위해 퍼시스턴트볼륨(PV) 및 퍼시스턴트볼륨클레임(PVC)이라는 두 가지 새로운 API 리소스를 제공합니다.

_퍼시스턴트볼륨_ (PV)은 관리자가 프로비저닝하거나 [스토리지 클래스](https://kubernetes.io/ko/docs/concepts/storage/storage-classes/)를 사용하여 동적으로 프로비저닝한 클러스터의 스토리지입니다. PV는 Volumes와 같은 볼륨 플러그인이지만, PV를 사용하는 개별 파드와는 별개의 라이프사이클을 가진다. 이 API 오브젝트는 NFS, iSCSI 또는 클라우드 공급자별 스토리지 시스템 등 스토리지 구현에 대한 세부 정보를 포함하고 있습니다.

_퍼시스턴트볼륨클레임_ (PVC)은 사용자의 스토리지에 대한 요청입니다. 이것은 파드와 비슷하다. 파드는 노드 리소스를 사용하고 PVC는 PV 리소스를 사용한다. 파드는 특정 수준의 리소스(CPU 및 메모리)를 요청할 수 있지만, 클레임은 특정 디스크 크기 및 접근 모드를 요청할 수 있습니다. (예: ReadWriteOnce, ReadOnlyMany 또는 ReadWriteMany로 마운트 할 수 있음. [AccessModes](https://kubernetes.io/ko/docs/concepts/storage/persistent-volumes/#%EC%A0%91%EA%B7%BC-%EB%AA%A8%EB%93%9C) 참고).

퍼시스턴트볼륨클레임을 사용하면 사용자가 추상화된 스토리지 리소스를 사용할 수 있지만, 다른 문제들 때문에 성능과 같은 다양한 속성을 가진 퍼시스턴트볼륨이 필요한 경우가 일반적입니다. 클러스터 관리자는 사용자에게 해당 볼륨의 구현 방법에 대한 세부 정보를 제공하지 않고 크기와 접근 모드와는 다른 방식으로 다양한 퍼시스턴트볼륨을 제공할 수 있어야 하며, 이러한 요구를 위해서는 _스토리지클래스_ 리소스가 있습니다.



### 3.1 스토리지 클래스 <a href="#storage-classes" id="storage-classes"></a>

클러스터에 이미 있는 스토리지 클래스를 확인합니다.

```
kubectl get storageclass

```

다음과 같은 결과를 확인 할 수 있습니다.

```
$ kubectl get storageclass
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  18h
```

최초 Worker Node를 배포할 때 생성한 기본 Storage Class 입니다.

### 3.2. Amazon EBS CSI 드라이버 구성

> 참조 URL : [https://docs.aws.amazon.com/ko\_kr/eks/latest/userguide/ebs-csi.html](https://docs.aws.amazon.com/ko\_kr/eks/latest/userguide/ebs-csi.html)

클러스터를 처음 생성할 때 Amazon EBS CSI 드라이버가 설치되지 않습니다. 드라이버를 사용하려면 Amazon EKS 추가 기능 또는 자체 관리형 추가 기능으로 드라이버를 추가해야 합니다.

Amazon EBS CSI 드라이버 구성을 위해서는 아래와 같은 내용을 참고합니다.

* Amazon EBS CSI 플러그 인이 사용자를 대신하여 AWS API를 호출하려면 IAM 권한이 필요합니다.
* Fargate에서 Amazon EBS CSI 컨트롤러를 실행할 수 있지만 Fargate pods에 볼륨을 탑재할 수는 없습니다.
* Amazon EKS 클러스터에서는 Amazon EBS CSI 드라이버의 알파 기능을 지원하지 않습니다.

### 3.3 Service Account 대한 EBS CSI 드라이버 IAM 역할 생성 <a href="#csi-iam-role" id="csi-iam-role"></a>

Amazon EBS CSI 플러그 인이 사용자를 대신하여 AWS API를 호출하려면 IAM 권한이 필요합니다.

플러그 인이 배포되면 `ebs-csi-controller-sa`라는 서비스 계정을 생성하고 해당 계정을 사용하도록 구성됩니다. 이 서비스 계정은 필요한 Kubernetes 권한이 할당되어 있는 Kubernetes `clusterrole`에 바인딩됩니다.

Service Account에 대한 EBS CSI 드라이버 IAM Role 생성과 연계를 위해서 OIDC(Open ID Connection)이 구성되어 있어야 합니다.

```
eksctl utils associate-iam-oidc-provider \
    --cluster ${ekscluster_name} \
    --approve

```

`eksctl`을 사용하여 Amazon EBS CSI 플러그 인 IAM 역할 생성합니다.

```
#eksctl을 사용하여 Amazon EBS CSI 플러그 인 IAM 역할 생성
eksctl create iamserviceaccount \
  --region ${AWS_REGION}\
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster ${ekscluster_name} \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --role-only \
  --role-name AmazonEKS_EBS_CSI_DriverRole
 
```

### 3.4 EBS Volume 암호화

Amazon EBS 볼륨의 암호화에 사용자 지정 [KMS 키](http://aws.amazon.com/kms/)를 사용하는 경우 필요에 따라 IAM 역할을 사용자 지정합니다.

앞서 우리는 KMS Key를 사전에 정의했습니다.

```
#KMS ARN 확인
echo ${MASTER_ARN}

```

EBS 볼륨을 생성할 때 암호화를 위해, IAM 역할에 대한 사용자 지정을 위해 아래와 같이 정책을 생성합니다.

```
#KMS 기반의 EBS 볼륨 암호화를 위한 정책 JSON 파일 생성.

mkdir ~/environment/secrete
cat <<EoF > ~/environment/secrete/kms-key-for-encryption-on-ebs.yaml
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kms:CreateGrant",
        "kms:ListGrants",
        "kms:RevokeGrant"
      ],
      "Resource": ["${MASTER_ARN}"],
      "Condition": {
        "Bool": {
          "kms:GrantIsForAWSResource": "true"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource": ["${MASTER_ARN}"]
    }
  ]
}
EoF

```

생성된 Policy (정책) JSON 파일을 IAM에 생성합니다.

```
aws iam create-policy \
  --policy-name KMS_Key_For_Encryption_On_EBS_Policy \
  --policy-document file://~/environment/secrete/kms-key-for-encryption-on-ebs.yaml

```

이제 정책을 연결합니다.

```
aws iam attach-role-policy \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/KMS_Key_For_Encryption_On_EBS_Policy \
  --role-name AmazonEKS_EBS_CSI_DriverRole

```



### 3.5 Amazon EKS Addon으로 Amazon EBS CSI 드라이버 관리 <a href="#managing-ebs-csi" id="managing-ebs-csi"></a>

보안을 강화하고 작업량을 줄이려면 Amazon EKS CSI 드라이버를 Amazon EKS 추가 기능으로 관리할 수 있습니다.

{% hint style="info" %}
Amazon EBS CSI 드라이버의 스냅샷 기능을 사용하려면 추가 기능을 설치하기 전에 스냅샷을 설치해야 합니다.&#x20;
{% endhint %}

Snapshot을 설치해야 한다면 아래 명령을 추가하고, Addon 작업을 진행합니다.

```
# volumesnapshotclasses, volumesnapshots 및 volumesnapshotcontent 에 대한 CRD설치
# RBAC, Controller 배포
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml

```

`eksctl`을 사용하여 Amazon EBS CSI 추가 기능을 추가하려면 다음 명령을 실행합니다.

```
# EBS CSI Addon 명령 실행
eksctl create addon --region ${AWS_REGION} --name aws-ebs-csi-driver --cluster ${ekscluster_name}  --service-account-role-arn arn:aws:iam::${ACCOUNT_ID}:role/AmazonEKS_EBS_CSI_DriverRole --force

```

EKS 메뉴의 Add-ons(추가기능)을 확인하면, 아래와 같이 정상적으로 EBS CSI Driver가 추가된 것을 확인 할 수 있습니다.

<figure><img src="../.gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>

Amazon EBS CSI 추가 기능의 최신 버전을 확인합니다.

```
eksctl get addon --name aws-ebs-csi-driver --cluster ${ekscluster_name}

```

최신버전으로 업데이트하려면, **`UPDATE AVAILABLE`** 에서 출력된 값으로 업데이트를 시도합니다.

아래 예제는 **`v1.18.0-eksbuild.1`** 로 업그레이드하는 예제입니다.

```
#Addon 기반 업데이트
export EBS_DRIVER_UPDATE_VERSION=v1.18.0-eksbuild.1
eksctl update addon --name aws-ebs-csi-driver --version ${EBS_DRIVER_UPDATE_VERSION} --cluster ${ekscluster_name} --force

```









