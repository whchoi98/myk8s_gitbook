# 인증 및 자격증명 환경 구성

## 인증 및 자격증명 환경 구성

### 1.IAM 역할 생성

**IAM - 역할 - 역할 만들기**

IAM 서비스 대쉬보드 접속 및 역할\(Role\) 생성을 합니다. AWS 서비스에서 IAM을 선택하고, "역할 만들기"를 선택합니다.

![](../.gitbook/assets/image%20%288%29.png)

### 2.역할\(Role\)을 만듭니다.

EC2 - 다음: 권한

역할 만들기 단계에서 EC2를 선택하고, "다음:권한"을 선택합니다.

![](../.gitbook/assets/image%20%282%29.png)

권한정책 연결에서 "AdministartorAccess"를 선택합니다.

![](../.gitbook/assets/image%20%285%29.png)

역할 이름 생성 및 정책 연결을 확인합니다.

역할 이름

```text
eksworkshop-admin
```

![](../.gitbook/assets/image%20%287%29.png)

### 3.Cloud9 권한 설정.

Cloud9 상단 좌측 메뉴에서 EC2 대쉬보드로 접속을 선택하거나, AWS 서비스에서 EC2 대쉬보드로 접속합니다.

![](../.gitbook/assets/image%20%286%29.png)

EC2 대쉬보드의 인스턴스에는 이미 생성된 Cloud9 EC2 인스턴스가 보입니다.

"작업"-"인스턴스 설정"-"IAM 역할 연결/바꾸기"를 선택합니다.

![](../.gitbook/assets/image%20%283%29.png)

앞서 생성한 IAM 역할 **"eksworkshop-admin"**을 선택합니다.

![](../.gitbook/assets/image%20%281%29.png)

정상적으로 Cloud9 IDE의 IAM 역할이 정상적으로 연결되었는지 확인합니다.

![](../.gitbook/assets/image%20%284%29.png)



{% hint style="warning" %}
Cloud9 생성에서 Network 설정을 하지 않았다면, Default VPC - Public Subnet에 Cloud 9 인스턴스는 배치됩니다.
{% endhint %}



