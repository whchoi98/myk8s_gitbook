---
description: 'Update : 2023-07-25'
---

# EKS Fargate

## EKS Fargate Overview

AWS Fargate는 컨테이너에 적절한 규모의 온디맨드 컴퓨팅 용량을 제공하는 기술입니다.&#x20;

AWS Fargate를 사용하면 더 이상 컨테이너를 실행하기 위해 가상 머신 그룹을 프로비저닝, 구성 또는 확장할 필요가 없습니다. 이렇게 하면 서버 유형을 선택하거나, 노드 그룹을 확장할 시기를 결정하거나, 클러스터 패킹을 최적화할 필요가 없습니다. Fargate에서 시작하는 포드와 Amazon EKS 클러스터의 일부로 정의된 Fargate 프로필로 실행되는 포드를 제어할 수 있습니다. 이 챕터에서는 EKS Fargate에 게임 2048 게임을 배포하고 Application Load Balancer를 사용하여 인터넷에 노출하는 방식을 구현해 봅니다.

관리자는 Fargate 프로필을 사용하여 Fargate에서 실행되는 포드를 선언할 수 있습니다.&#x20;

* 각 프로필에는 네임스페이스와 레이블이 포함된 Selector가 최대 5개 있을 수 있습니다.&#x20;
* 모든 Selector에 대해 네임스페이스를 정의해야 합니다.&#x20;
* 레이블 필드는 Key / Value로 구성됩니다. Selector가 일치하는 포드는 Fargate에서 예약됩니다. 일반적으로 사용자 애플리케이션 워크로드를 kube-system 또는 default 이외의 네임스페이스에 배포하여 EKS에 배포된 포드 간의 상호 작용을 관리하는 보다 세분화된 기능을 갖는 것이 좋습니다.
* &#x20;fargate 네임스페이스를 대상으로 하는 모든 포드를 대상으로 하는 application이라는 새 Fargate 프로필을 생성합니다.

Fargate 구성요소

*   포드 실행 역할 - 클러스터가 AWS Fargate에 Pods를 생성하는 경우 Fargate 인프라에서 실행 중인 `kubelet`이 사용자를 대신하여 AWS API를 호출해야 합니다. 예를 들어, Amazon ECR에서 컨테이너 이미지를 가져오기 위해 호출해야 합니다. Amazon EKS Pod 실행 역할은 이 작업을 수행할 수 있는 IAM 권한을 제공합니다.

    Fargate 프로파일을 생성할 때 Pod와 함께 사용할 Pods 실행 역할을 지정해야 합니다. 이 역할은 권한 부여를 위해 클러스터의 Kubernetes [역할 기반 액세스 컨트롤](https://kubernetes.io/docs/admin/authorization/rbac/) (RBAC)에 추가됩니다. 이렇게 하면 Fargate 인프라에서 실행 중인 `kubelet`이 Amazon EKS 클러스터에 등록되어 클러스터에 노드로 표시될 수 있습니다. 자세한 내용은 [Amazon EKS Pod 실행 IAM 역할](https://docs.aws.amazon.com/ko\_kr/eks/latest/userguide/pod-execution-role.html) 섹션을 참조하세요.
* 서브넷 - 이 프로파일을 사용하는 Pods를 시작할 서브넷의 ID입니다. 현재 Fargate에서 실행 중인 Pods에는 퍼블릭 IP 주소가 할당되지 않습니다. 따라서 이 파라미터에 대해서는 인터넷 게이트웨이로 가는 직접 경로가 없는 프라이빗 서브넷만 허용됩니다.
* 셀렉터 - 이 Fargate 프로파일을 사용하기 위해 Pods에 대해 매칭하는 선택기입니다. Fargate 프로파일에 선택기를 최대 5개까지 지정할 수 있습니다. 셀렉터에는 다음 구성 요소가 있습니다.
  * 네임스페이스 - 셀렉터에 대한 네임스페이스를 지정해야 합니다. 셀렉터는 이 네임스페이스에서 생성된 Pods만 일치시킵니다. 그러나 여러 셀렉터를 생성하여 여러 네임스페이스를 대상으로 지정할 수 있습니다.
  * 레이블 - 선택적으로 셀렉터에 대해 일치시킬 Kubernetes 레이블을 지정할 수 있습니다. 셀렉터는 셀렉터에 지정된 모든 레이블이 있는 Pods와 일치합니다.

## EKS Fargate 구성

### Fargate Profile 생성

아래와 같은 eksctl 명령을 통해서 Fargate Profile을 생성합니다.

<pre><code><strong>#eks fargate profile 생성
</strong><strong>eksctl create fargateprofile \
</strong>  --cluster $ekscluster_name \
  --name fargate-game-2048 \
  --namespace fargate-game-2048

</code></pre>

EKS 클러스터가 Fargate에서 포드를 예약할 때 포드는 Amazon ECR에서 컨테이너 이미지 가져오기와 같은 작업을 수행하기 위해 사용자를 대신하여 AWS API를 호출해야 합니다.&#x20;

Fargate 포드 실행 역할은 이를 수행할 수 있는 IAM 권한을 제공합니다. 이 IAM 역할은 위의 명령에 의해 자동으로 생성됩니다. Fargate 프로파일 생성에는 최대 몇 분이 소요될 수 있습니다. 프로파일 생성이 완료된 후 다음 명령을 실행합니다.

```
eksctl get fargateprofile \
  --cluster $ekscluster_name \
  -o yaml
  
```

아래와 같은 유사한 출력 결과를 얻으실 수 있습니다.

```
- name: fargate-game-2048
  podExecutionRoleARN: arn:aws:iam::410691154611:role/eksctl-eksworkshop-fargate-FargatePodExecutionRole-VC3M4TH43PN2
  selectors:
  - namespace: fargate-game-2048
  status: ACTIVE
  subnets:
  - subnet-044fd202626076e79
  - subnet-046e417018b50d939
  - subnet-0d3c2d117533c431d
```

EKS workshop에서 이미 앞서 subnet에 대한 정보를 변수로 저장해 두었습니다.

cat \~/.bash\_profile에서 어느 서브넷에 배치되었는지 확인해 봅니다. EKS 클러스터의 프라이빗 서브넷이 포함되어 있습니다. Fargate에서 실행되는 포드에는 퍼블릭 IP 주소가 할당되지 않으므로 Fargate 프로파일을 생성할 때 프라이빗 서브넷만 지원됩니다.&#x20;

따라서 EKS 클러스터를 프로비저닝하는 동안 생성하는 VPC에 하나 이상의 프라이빗 서브넷이 포함되어 있는지 확인해야 합니다. LAB에서는 3개의 Public Subnet과 3개의 Private Subnet을 할당했기 때문에, Private Subnet 3개에 할당된 것을 알 수 있습니다.

```
$ cat ~/.bash_profile | grep Private
export PrivateSubnet01=subnet-044fd202626076e79
export PrivateSubnet02=subnet-046e417018b50d939
export PrivateSubnet03=subnet-0d3c2d117533c431d

```

### Application 배포

2048 게임 어플리케이션과 Ingress Gateway (AWS Loadbalancer Controller)를 생성해서 외부에 서비스를 제공해 봅니다.

ALB Loadbalancer Controller 기반의 서비스를 제공하기 위해서는 사전에 AWS LBC(AWS LoadBalancer Controller)가 설치되어 있어야 합니다. (참조 - [AWS Load Balancer Controller](../5.eks-ingress/alb-ingress.md))

fargate\_2048\_full.yaml 파일은 앞서 git에서 복제를 해 두었습니다.

```
kubectl apply -f ~/environment/myeks/alb-controller/fargate_2048_full.yaml 
kubectl -n fargate-game-2048 rollout status deployment deployment-2048

```

아래 명령을 통해서, 현재 생성된 Node 숫자를 확인해 봅니다.

```
kubectl get nodes -o wide

```

아래와 같은 결과를 확인해 볼 수 있습니다.

```
$ kubectl get nodes -o wide
NAME                                                      STATUS   ROLES    AGE     VERSION                INTERNAL-IP    EXTERNAL-IP     OS-IMAGE         KERNEL-VERSION                  CONTAINER-RUNTIME
fargate-ip-10-11-64-72.ap-northeast-2.compute.internal    Ready    <none>   6m27s   v1.23.17-eks-f4dc2c0   10.11.64.72    <none>          Amazon Linux 2   5.10.184-175.731.amzn2.x86_64   containerd://1.6.6
fargate-ip-10-11-69-151.ap-northeast-2.compute.internal   Ready    <none>   6m18s   v1.23.17-eks-f4dc2c0   10.11.69.151   <none>          Amazon Linux 2   5.10.184-175.731.amzn2.x86_64   containerd://1.6.6
fargate-ip-10-11-79-252.ap-northeast-2.compute.internal   Ready    <none>   6m26s   v1.23.17-eks-f4dc2c0   10.11.79.252   <none>          Amazon Linux 2   5.10.184-175.731.amzn2.x86_64   containerd://1.6.6
fargate-ip-10-11-85-67.ap-northeast-2.compute.internal    Ready    <none>   6m27s   v1.23.17-eks-f4dc2c0   10.11.85.67    <none>          Amazon Linux 2   5.10.184-175.731.amzn2.x86_64   containerd://1.6.6
fargate-ip-10-11-93-229.ap-northeast-2.compute.internal   Ready    <none>   6m29s   v1.23.17-eks-f4dc2c0   10.11.93.229   <none>          Amazon Linux 2   5.10.184-175.731.amzn2.x86_64   containerd://1.6.6
ip-10-11-1-156.ap-northeast-2.compute.internal            Ready    <none>   32h     v1.23.17-eks-a5565ad   10.11.1.156    3.35.137.161    Amazon Linux 2   5.4.247-162.350.amzn2.x86_64    docker://20.10.23
ip-10-11-11-192.ap-northeast-2.compute.internal           Ready    <none>   32h     v1.23.17-eks-a5565ad   10.11.11.192   13.125.150.9    Amazon Linux 2   5.4.247-162.350.amzn2.x86_64    docker://20.10.23
ip-10-11-22-97.ap-northeast-2.compute.internal            Ready    <none>   32h     v1.23.17-eks-a5565ad   10.11.22.97    15.164.178.80   Amazon Linux 2   5.4.247-162.350.amzn2.x86_64    docker://20.10.23
ip-10-11-27-37.ap-northeast-2.compute.internal            Ready    <none>   32h     v1.23.17-eks-a5565ad   10.11.27.37    15.165.12.198   Amazon Linux 2   5.4.247-162.350.amzn2.x86_64    docker://20.10.23
이하 생략
```

{% hint style="info" %}
EKS Fargate는 1개의 Pod가 1개의 Node를 차지합니다. fargate\_2048\_full.yaml에는 5개의 Pod가 구동하도록 replica가 세팅되어 있습니다. 따라서 5개의 Node가 Private Subnet에서 동작하게 됩니다.
{% endhint %}

아래 명령을 통해서 replica를 줄이고, 다시 node 숫자를 확인해 봅니다.

```
#Pod가 배포된 Node를 확인
kubectl -n fargate-game-2048 get pod -o wide

#Fargate Replica 숫자를 줄여 봅니다.
kubectl -n fargate-game-2048 deployment fargate-game-2048 --replicas=3
kubectl -n fargate-game-2048 get pod -o wide
kubectl get nodes -o wide

```

아래와 같이 변경될 결과를 확인해 볼 수 있습니다.

```
 $ kubectl -n fargate-game-2048 scale deployment deployment-2048 --replicas=3
deployment.apps/deployment-2048 scaled

 $ kubectl -n fargate-game-2048 get pod -o wide
NAME                               READY   STATUS    RESTARTS   AGE   IP             NODE                                                      NOMINATED NODE   READINESS GATES
deployment-2048-69d8c56689-5dsc2   1/1     Running   0          18m   10.11.79.252   fargate-ip-10-11-79-252.ap-northeast-2.compute.internal   <none>           <none>
deployment-2048-69d8c56689-8djvg   1/1     Running   0          18m   10.11.93.229   fargate-ip-10-11-93-229.ap-northeast-2.compute.internal   <none>           <none>
deployment-2048-69d8c56689-ds228   1/1     Running   0          18m   10.11.64.72    fargate-ip-10-11-64-72.ap-northeast-2.compute.internal    <none>           <none>

$ kubectl get nodes -o wide
NAME                                                      STATUS   ROLES    AGE   VERSION                INTERNAL-IP    EXTERNAL-IP     OS-IMAGE         KERNEL-VERSION                  CONTAINER-RUNTIME
fargate-ip-10-11-64-72.ap-northeast-2.compute.internal    Ready    <none>   17m   v1.23.17-eks-f4dc2c0   10.11.64.72    <none>          Amazon Linux 2   5.10.184-175.731.amzn2.x86_64   containerd://1.6.6
fargate-ip-10-11-79-252.ap-northeast-2.compute.internal   Ready    <none>   17m   v1.23.17-eks-f4dc2c0   10.11.79.252   <none>          Amazon Linux 2   5.10.184-175.731.amzn2.x86_64   containerd://1.6.6
fargate-ip-10-11-93-229.ap-northeast-2.compute.internal   Ready    <none>   17m   v1.23.17-eks-f4dc2c0   10.11.93.229   <none>          Amazon Linux 2   5.10.184-175.731.amzn2.x86_64   containerd://1.6.6

```

EKS Fargate는 완전관리형 형태로 제공되기 때문에, EC2 대시보드에서 확인할 수 없습니다.

Amazon Elastic Kubernetes Service 의 **`Cluster - Computing`** 대시보드에서 확인이 가능합니다.

<figure><img src="../.gitbook/assets/image (303).png" alt=""><figcaption></figcaption></figure>

AWS Fargate 고려 사항

Amazon EKS에서 Fargate를 사용할 때 고려해야 할 몇 가지 사항이 있습니다.

* Fargate에서 실행되는 각 Pod에는 자체 격리 경계가 있습니다. 기본 커널, CPU 리소스, 메모리 리소스 또는 Elastic 네트워크 인터페이스를 다른 Pod와 공유하지 않습니다.
* Network Load Balancer 및 Application Load Balancer(ALB)는 IP Target을 통해서만 Fargate에서 사용할 수 있습니다.&#x20;
* DaemonSets는 Fargate에서 지원되지 않습니다. 애플리케이션에 Daemon이 필요한 경우 Pods에서 사이드카 컨테이너로 실행되도록 해당 Daemon을 다시 구성합니다.
* 권한 있는 컨테이너(Privileged container)는 Fargate에서 지원되지 않습니다.
* Fargate에서 실행되는 포드는 Pod 매니페스트에서 `HostPort` 또는 `HostNetwork`를 지정할 수 없습니다.
* Fargate Pods의 경우 기본 `nofile` 및 `nproc` 소프트 제한은 1,024이고 하드 제한은 65,535입니다.
* GPU는 현재 Fargate에서 사용할 수 없습니다.
* Fargate에서 실행되는 포드는 인터넷 게이트웨이에 대한 직접 경로가 없고 AWS 서비스에 대한 NAT 게이트웨이 액세스 권한이 있는 프라이빗 서브넷에서만 지원되므로 클러스터의 VPC에 프라이빗 서브넷이 있어야 합니다. 아웃바운드 인터넷 액세스가 없는 클러스터의 경우 [프라이빗 클러스터 요구 사항](https://docs.aws.amazon.com/ko\_kr/eks/latest/userguide/private-clusters.html)을 참조하십시오.
* [Vertical Pod Autoscaler](https://docs.aws.amazon.com/ko\_kr/eks/latest/userguide/vertical-pod-autoscaler.html)를 사용하여 Fargate Pods의 올바른 초기 CPU 및 메모리 크기를 설정한 다음 [Horizontal Pod Autoscaler](https://docs.aws.amazon.com/ko\_kr/eks/latest/userguide/horizontal-pod-autoscaler.html)를 사용하여 해당 Pods를 조정할 수 있습니다. Vertical Pod Autoscaler가 더 큰 CPU 및 메모리 조합으로 Pods를 Fargate에 자동으로 다시 배포하도록 하려면 Vertical Pod Autoscaler의 모드를 `Auto` 또는 `Recreate`로 설정하여 올바로 작동하게 해야 합니다. 자세한 내용은 GitHub에서 [Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler#quick-start)설명서를 참조하세요.
* VPC에 대해 DNS 확인 및 DNS 호스트 이름을 활성화해야 합니다. 자세한 내용은 [VPC에 대한 DNS 지원 보기 및 업데이트](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-dns.html#vpc-dns-updating)를 참조하십시오.
*   Amazon EKS Fargate는 가상 머신(VM) 내에서 각 포드를 격리하여 Kubernetes 애플리케이션에 대한 심층 방어 기능을 추가합니다. 이 VM 경계를 통해 컨테이너 이스케이프 시 다른 포드에서 사용하는 호스트 기반 리소스에 대한 액세스를 방지하는데, 이 방법은 컨테이너화된 애플리케이션을 공격하고 컨테이너 외부의 리소스에 대한 액세스 권한을 얻는 일반적인 방법입니다.

    Amazon EKS를 사용해도 [공동 책임 모델](https://docs.aws.amazon.com/ko\_kr/eks/latest/userguide/security.html)에서 책임은 변경되지 않습니다. 클러스터 보안 및 거버넌스 제어의 구성을 신중하게 고려해야 합니다. 애플리케이션을 격리하는 가장 안전한 방법은 항상 별도의 클러스터에서 실행하는 것입니다.
* Fargate 프로필은 VPC 보조 CIDR 블록의 서브넷 지정을 지원합니다. 보조 CIDR 블록을 지정할 수도 있습니다. 서브넷에서 사용 가능한 IP 주소의 수는 제한되어 있기 때문입니다. 따라서 클러스터에서 생성할 수 있는 Pods의 수가 제한되어 있습니다. Pods에 다른 서브넷을 사용하여 사용 가능한 IP 주소의 수를 늘릴 수 있습니다. 자세한 내용을 알아보려면 [VPC에 `IPv4` CIDR 블록 추가](https://docs.aws.amazon.com/vpc/latest/userguide/VPC\_Subnets.html#vpc-resize)를 참조하세요.
* Amazon EC2 인스턴스 메타데이터 서비스(IMDS)는 Fargate 노드에 배포된 Pods에는 사용할 수 없습니다. IAM 자격 증명이 필요한 Pods를 Fargate에 배포한 경우 [서비스 계정에 대한 IAM 역할](https://docs.aws.amazon.com/ko\_kr/eks/latest/userguide/iam-roles-for-service-accounts.html)을 사용하여 Pods에 할당합니다. Pods가 IMDS를 통해 제공되는 다른 정보에 액세스해야 할 경우 이 정보를 Pod 사양에 하드 코딩해야 합니다. 여기에는 Pod가 배포된 AWS 리전 또는 가용 영역이 포함됩니다.
* Fargate Pods는 AWS Outposts, AWS Wavelength 또는 AWS 로컬 영역에 배포할 수 없습니다.
* Amazon EKS는 정기적으로 Fargate Pods를 패치하여 보안을 유지해야 합니다. 영향을 줄이는 방식으로 업데이트를 시도하지만 Pods가 성공적으로 제거되지 않으면 삭제해야 하는 경우가 있습니다. 중단을 최소화하기 위해 취할 수 있는 몇 가지 조치가 있습니다. 자세한 내용은 [Fargate OS 패치](https://docs.aws.amazon.com/ko\_kr/eks/latest/userguide/fargate-pod-patching.html) 섹션을 참조하세요.
* [Amazon EKS용 Amazon VPC CNI 플러그인](https://github.com/aws/amazon-vpc-cni-plugins)은 Fargate 노드에 설치됩니다. Fargate 노드에서는 [호환 가능한 대체 CNI 플러그 인](https://docs.aws.amazon.com/ko\_kr/eks/latest/userguide/alternate-cni-plugins.html)을 사용할 수 없습니다.
* Fargate에서 Amazon EBS CSI 컨트롤러를 실행할 수 있지만 Fargate Pods에 볼륨을 탑재할 수는 없습니다.
