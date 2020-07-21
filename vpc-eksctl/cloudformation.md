# Cloudformation 구성

## Cloudformation 소개

AWS CloudFormation에서는 클라우드 환경에서 AWS 및 타사 애플리케이션 리소스를 모델링하고 프로비저닝할 수 있도록 공용 언어를 제공합니다. AWS CloudFormation을 사용하면 프로그래밍 언어 또는 간단한 텍스트 파일을 사용하여 자동화되고 안전한 방식으로 모든 지역과 계정에 걸쳐 애플리케이션에 필요한 모든 리소스를 모델링 및 프로비저닝할 수 있습니다. 이를 통해 AWS 및 타사 리소스의 단일 소스를 제공합니다.

## Cloudformation 기반의 VPC 구성

### 1.VPC yaml 다운로드

eksworkshop에서 사용할 vpc yaml을 다운로드 받습니다.

```text
https://github.com/whchoi98/myeks/blob/master/EKSVPC.yml
```

### 2. Stack 생성

AWS 서비스 - Cloudformation 을 선택합니다. 

"스택생성" - "새 리소스 사용"을 선택합니다.

![](../.gitbook/assets/image%20%2830%29.png)

앞서 다운로드 받은 EKSVPC.yml 파일을 선택해서 업로드합니다.

![](../.gitbook/assets/image%20%2817%29.png)

### 3. Stack 상세 정보 구성

Cloudformation Stack 상세 정보 구성을 합니다.

![](../.gitbook/assets/image%20%2818%29.png)

![](../.gitbook/assets/image%20%285%29.png)

생성이 완료되었는지 확인합니다.

![](../.gitbook/assets/image%20%2826%29.png)

4.Output 정보 확인

Cloudformation을 통해 생성된 VPC의 자원들을 기반으로, eksctl 을 사용해서 EKS Cluster를 사용할 것입니다.

이때 필요한 것이, VPC id, Subnet id 입니다. 출력이 되면 subnet id는 public subnet 01,02,03, private subnet 01,02,03으로 출력됩니다. 

이 값을 따로 저장해 둡니다.

![](../.gitbook/assets/image%20%2829%29.png)

