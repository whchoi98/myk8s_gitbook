---
description: 'Update : 2022-06-01 / 10min'
---

# EKS 구성확인

## EKS Cluster 구성 확인&#x20;

EKS 콘솔을 통해서, 생성된 EKS Cluster를 확인 할 수 있습니다.

각 계정의 User로 로그인 한 경우, 아래에서 처럼 eks cluster에 대한 정보를 확인 할 수 없습니다. 이것은 권한이 없기 때문입니다. User의 권한을 Cloud9에서 추가해 줍니다. &#x20;

![](<../.gitbook/assets/image (176).png>)

## configmap 인증 정보 수정

cloud9 IDE Terminal 에 kubectl 명령을 통해서, 계정의 사용자에 대한 권한을 추가해 줍니다.

```
kubectl edit -n kube-system configmap/aws-auth
```

user arn은 IAM에서 확인 할 수 있습니다.&#x20;

**`AWS 관리 콘솔 - IAM - User`**

![](<../.gitbook/assets/image (170).png>)

아래 새로운 사용자의 권한을 mapRoles 뒤에 추가해 줍니다. kubectl edit는 vi edit과 동일하게 수정하는 방식입니다. 아래 명령을 입력하고 복사해서 붙여 넣습니다.

```
#vi 실행 창에 입력합니다
:set paste
```

```
  mapUsers: |
    - userarn: arn:aws:iam::xxxxxxxx:user/{username}
      username: {username}
      groups:
        - system:masters
```

추가한 이후 configmap/aws-auth 입니다.

```
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::584172017494:role/eksctl-eksworkshop-nodegroup-mana-NodeInstanceRole-1VEXSSMR1CWJ0
      username: system:node:{{EC2PrivateDNSName}}
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::584172017494:role/eksctl-eksworkshop-nodegroup-mana-NodeInstanceRole-1SYQ7SFAJFYQX
      username: system:node:{{EC2PrivateDNSName}}
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::584172017494:role/eksctl-eksworkshop-nodegroup-ng-p-NodeInstanceRole-S7BXB5A9FVBG
      username: system:node:{{EC2PrivateDNSName}}
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::584172017494:role/eksctl-eksworkshop-nodegroup-ng-p-NodeInstanceRole-83YEOE9PLC79
      username: system:node:{{EC2PrivateDNSName}}
  mapUsers: |
    - userarn: arn:aws:iam::584172017494:user/whchoi
      username: whchoi
      groups:
        - system:masters
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
  rolearn: arn:aws:iam::584172017494:role/eksctl-eksworkshop-nodegroup-mana-NodeInstanceRole-1VEXSSMR1CWJ0
  username: system:node:{{EC2PrivateDNSName}}
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:aws:iam::584172017494:role/eksctl-eksworkshop-nodegroup-mana-NodeInstanceRole-1SYQ7SFAJFYQX
  username: system:node:{{EC2PrivateDNSName}}
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:aws:iam::584172017494:role/eksctl-eksworkshop-nodegroup-ng-p-NodeInstanceRole-S7BXB5A9FVBG
  username: system:node:{{EC2PrivateDNSName}}
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:aws:iam::584172017494:role/eksctl-eksworkshop-nodegroup-ng-p-NodeInstanceRole-83YEOE9PLC79
  username: system:node:{{EC2PrivateDNSName}}

mapUsers:
----
- userarn: arn:aws:iam::584172017494:user/whchoi98
  username: whchoi98
  groups:
    - system:masters
```

## EKS Cluster 결과 확인.

EKS Cluster를 다시 콘솔에서 확인해 봅니다. 생성한 모든 노드들을 확인할 수 있습니다. 생성한 클러스터를 선택합니다

![](<../.gitbook/assets/image (219).png>)

컴퓨팅을 선택하고, 생성된 WorkerNode들을 확인해 봅니다

![](<../.gitbook/assets/image (234).png>)

EKS Cluster내에 생성된 워크로드들을 확인해 볼 수 있습니다.

![](<../.gitbook/assets/image (236).png>)

managed Node type과 Self Managed Node Type의 차이를 확인할 수 있습니다

Kuernetes의 Resource들을 선택하고 확인해 봅니다.&#x20;

![](<../.gitbook/assets/image (237).png>)

이제 아래와 같은 EKS Cluster가 완성되었습니다. kubectl 명령을 통해 확인해 봅니다.

```
#kube-system namespace에 생성된 자원 확인 
kubectl -n kube-system get all

#주요 Pod의 상세 정보 확인 
kubectl -n kube-system pods <pod-name> -o wide

# node 상세 정보 확인 
kubectl get nodes -o wide

```

![](<../.gitbook/assets/image (180).png>)

다음 섹션에서 워크로드들을 생성한 이후 EKS Cluster 메뉴에서 확인해 봅니다.

