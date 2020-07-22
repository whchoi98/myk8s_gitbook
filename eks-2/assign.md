# Assign

## Assign 소개. 

Pod의 효과적인 할당을 위한 기술 전략과 작동 방식 및 다양한 방법들에 대해 소개합니다.

필요에 따라 특정 노드에서만 실행하거나 특정 노드에서 실행하는 것에 우선 순위를 두도록 제한할 수 있습니다. 

스케쥴러가 자동으로 적절한 배치를 수행하므로 일반적으로 이러한 제한들은 필요하지 않지만, 더 세밀한 제어가 필요한 경우가 프로덕션에서 발생할 수 있습니다.  

이번 실습에서는 이러한 요구 조건에 맞추어 Pod에 대한 할당을 중점적으로 다룹니다.

## NodeSelector

NodeSelector는 가장 간단한 노드 선택 방법입니다. NodeSelector는 PodSpec 필드에서 선택이 가능하며,  Key-value 페어로 지정하게 됩니다. Pod가 Node에 실행 될 수 있으려면, Node에 표시된 Key-Value 형태의 Lable이 존재해야 합니다. 이것은 다중으로 지정할 수도 있습니다.

이미 yaml로 정의된 EKS LAB을 eksctl로 배포시에, 각 Worker Node들에 Lable이 정의 되어 있었습니다.

```text
cat ~/environment/myeks/whchoi-cluster.yaml
```

whchoi-cluster.yaml의 내용을 살펴 봅니다.생

```text

nodeGroups:
  - name: ng1-public
    instanceType: m5.xlarge
    desiredCapacity: 3
    minSize: 3
    maxSize: 9
    volumeSize: 30
    volumeType: gp2 
    amiFamily: AmazonLinux2
    labels:
      nodegroup-type: "frontend-workloads"
    ssh: 
        publicKeyPath: "/home/ec2-user/environment/eksworkshop.pub"
        allow: true
    iam:
      attachPolicyARNs:
      withAddonPolicies:
        autoScaler: true
        cloudWatch: true
        ebs: true
        fsx: true
        efs: true

  - name: ng2-private
    instanceType: m5.xlarge
    desiredCapacity: 3
    privateNetworking: true
    minSize: 3
    maxSize: 9
    volumeSize: 30
    volumeType: gp2 
    amiFamily: AmazonLinux2
    labels:
      nodegroup-type: "backend-workloads"
    ssh: 
        publicKeyPath: "/home/ec2-user/environment/eksworkshop.pub"
        allow: true
    iam:
      withAddonPolicies:
        autoScaler: true
        cloudWatch: true
        ebs: true
        fsx: true
        efs: true


cloudWatch:
    clusterLogging:
        enableTypes: ["api", "audit", "authenticator", "controllerManager", "scheduler"]

```

## Affinity 와 Anti-Affinity

## 다양한 사례.



