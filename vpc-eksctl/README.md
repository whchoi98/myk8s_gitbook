---
description: 'Update : 2022-09-10 / 20min'
---

# 3.VPC구성과 eksctl 기반 설치

아래와 같은 구성을 Cloudformation과 eksctl기반으로 설치합니다.

![](<../.gitbook/assets/image (244).png>)

Cloudformation으로 VPC, AZ, Subnet, Routing Table등을 구성합니다.

eksctl로 사전에 정의된 yaml을 통해서 각 Public Node Group과 Private Node Group을 구성합니다.

이 랩을 완료하는데 1시간 정도 소요됩니다.&#x20;

