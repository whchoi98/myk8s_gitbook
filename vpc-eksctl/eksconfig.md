---
description: 'Update : 2022-09-10 / 10min'
---

# EKS 구성확인

## EKS Cluster 구성 확인&#x20;

EKS 콘솔을 통해서, 생성된 EKS Cluster를 확인 할 수 있습니다.

각 계정의 User로 로그인 한 경우, 아래에서 처럼 eks cluster에 대한 정보를 확인 할 수 없습니다. 이것은 권한이 없기 때문입니다. User의 권한을 Cloud9에서 추가해 줍니다. &#x20;

![](<../.gitbook/assets/image (377).png>)

## configmap 인증 정보 수정

cloud9 IDE Terminal 에 kubectl 명령을 통해서, aws-auth 파일을 확인해 봅니다.&#x20;

```
kubectl get configmap -n kube-system aws-auth -o yaml
```

aws-auth.yaml 파일을 아래 디렉토리에 생성합니다.&#x20;

```
kubectl get configmap -n kube-system aws-auth -o yaml | grep -v "creationTimestamp\|resourceVersion\|selfLink\|uid" | sed '/^  annotations:/,+2 d' > ~/environment/aws-auth.yaml
cp ~/environment/aws-auth.yaml ~/environment/aws-auth_backup.yaml

```

LAB에서 사용 중인 USER ID를 Shell 환경 변수에 입력하고, 현재 ACCOUNT ID를 확인합니다.

Account ID는 이미 Shell 환경변수에 저장해 두었습니다.

```
## USER_ID의 값은 현재 콘솔에서의 user id
export USER_ID=user01
echo "export USER_ID=${USER_ID}" | tee -a ~/.bash_profile
source ~/.bash_profile

## ACCOUNT ID 확인
echo $ACCOUNT_ID

```

eksctl 명령의 iamidentitymapping 을 사용해서, IAM user와 Kubernetes의 각 Role Binding 에 설정을 매핑합니다.

아래 예제에서는 user01을 system-masters의 role에 바인딩해서, 매핑합니다.

```
eksctl create iamidentitymapping \
  --cluster ${ekscluster_name} \
  --arn arn:aws:iam::${ACCOUNT_ID}:user/${USER_ID}\
  --username ${USER_ID} \
  --group system:masters
  
```

추가한 이후 aws-auth.yaml 파일 입니다.

```
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::027268078051:role/eksctl-eksworkshop-nodegroup-mana-NodeInstanceRole-1WKNHHYJD99CR
      username: system:node:{{EC2PrivateDNSName}}
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::027268078051:role/eksctl-eksworkshop-nodegroup-mana-NodeInstanceRole-WBJOTQC4HJOD
      username: system:node:{{EC2PrivateDNSName}}
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::027268078051:role/eksctl-eksworkshop-nodegroup-ng-p-NodeInstanceRole-122GCV62TF8AS
      username: system:node:{{EC2PrivateDNSName}}
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::027268078051:role/eksctl-eksworkshop-nodegroup-ng-p-NodeInstanceRole-1SRFNRXCSDVMW
      username: system:node:{{EC2PrivateDNSName}}
  mapUsers: |
    - userarn: arn:aws:iam::027268078051:user/user01
      username: user01
      groups:
        - system:masters
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
```

```
# 아래와 같이 터미널에서 직접 수정도 가능합니다. 
# kubectl edit -n kube-system configmap/aws-auth
```

실제 configmap에 대한 값을 확인해 봅니다. configmap은 kube-system namespace에 존재합니다.

```
kubectl describe configmap -n kube-system aws-auth
```

```
Name:         aws-auth
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>

Data
====
mapRoles:
----
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:aws:iam::027268078051:role/eksctl-eksworkshop-nodegroup-mana-NodeInstanceRole-1WKNHHYJD99CR
  username: system:node:{{EC2PrivateDNSName}}
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:aws:iam::027268078051:role/eksctl-eksworkshop-nodegroup-mana-NodeInstanceRole-WBJOTQC4HJOD
  username: system:node:{{EC2PrivateDNSName}}
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:aws:iam::027268078051:role/eksctl-eksworkshop-nodegroup-ng-p-NodeInstanceRole-122GCV62TF8AS
  username: system:node:{{EC2PrivateDNSName}}
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:aws:iam::027268078051:role/eksctl-eksworkshop-nodegroup-ng-p-NodeInstanceRole-1SRFNRXCSDVMW
  username: system:node:{{EC2PrivateDNSName}}

mapUsers:
----
- userarn: arn:aws:iam::027268078051:user/user01
  username: user01
  groups:
    - system:masters

Events:  <none>
```



아래와 같이 eksctl 명령을 통해 현재 IAM과 Kubernetes Role 이 바인딩 된 목록을 확인합니다.

```
eksctl get iamidentitymapping --cluster ${ekscluster_name}

```

## EKS Cluster 결과 확인.

EKS Cluster를 다시 콘솔에서 확인해 봅니다. 생성한 모든 노드들을 확인할 수 있습니다. 생성한 클러스터를 선택합니다

![](<../.gitbook/assets/image (350).png>)

컴퓨팅을 선택하고, 생성된 WorkerNode들을 확인해 봅니다

![](<../.gitbook/assets/image (67).png>)

EKS Cluster내에 생성된 워크로드들을 확인해 볼 수 있습니다.

![](<../.gitbook/assets/image (339).png>)

<figure><img src="../.gitbook/assets/image (158).png" alt=""><figcaption></figcaption></figure>

managed Node type과 Self Managed Node Type의 차이를 확인할 수 있습니다

Kuernetes의 Resource들을 선택하고 확인해 봅니다.&#x20;

![](<../.gitbook/assets/image (48).png>)

이제 아래와 같은 EKS Cluster가 완성되었습니다. kubectl 명령을 통해 확인해 봅니다.

```
#kube-system namespace에 생성된 자원 확인 
kubectl -n kube-system get all

#주요 Pod의 상세 정보 확인 
kubectl -n kube-system pods <pod-name> -o wide

# node 상세 정보 확인 
kubectl get nodes -o wide

```

![](<../.gitbook/assets/image (287).png>)

다음 섹션에서 워크로드들을 생성한 이후 EKS Cluster 메뉴에서 확인해 봅니다.

