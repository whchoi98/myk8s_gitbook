# Helm

## Helm 소개

헬름은 [쿠버네티스](https://kubernetes.io/)용 소프트웨어를 검색하거나, 공유하고 사용하기에 가장 좋은 방법입니다. 쿠버네티스 차트를 관리하기 위한 도구이며, 차트는 사전 구성된 쿠버네티스 리소스의 패키지입니다.

주요 3가지의 개념이 있습니다.

**차트**는 헬름 패키지입니. 이 패키지에는 쿠버네티스 클러스터 내에서 애플리게이션, 도구, 서비스를 구동하는데 필요한 모들 리소스 정의가 포함되어 있습니다. 쿠버네티스에서의 Homebrew 포뮬러, Apt dpkg, YUM RPM 파일과 같은 것으로 생각할 수 있습니다.

**저장소**는 차트를 모아두고 공유하는 장소입다. 이것은 마치 Perl의 [CPAN 아카이브](https://www.cpan.org/)나 [페도라 패키지 데이터베이스](https://admin.fedoraproject.org/pkgdb/)와 같은데, 쿠버네티스 패키지용이라고 보면 됩니다.

**릴리스**는 쿠버네티스 클러스터에서 구동되는 차트의 인스턴스입다. 일반적으로 하나의 차트는 동일한 클러스터내에 여러 번 설치될 수 있다. 설치될 때마다, 새로운 _release_ 가 생성됩니다.

헬름은 쿠버네티스 내부에 \_charts\_를 설치하고, 각 설치에 대해 새로운 \_release\_를 생성하고, 새로운 차트를 찾기 위해 헬름 차트 \_repositories\_를 검색할 수 있습니다.

## Helm 설치와 간단한 배포

### 1.Helm 설치

헬름은 헬름 최신 버전을 자동으로 가져와서 [로컬에 설치](https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3)하는 인스톨러 스크립트를 제공합니다. 이 스크립트를 받아서 로컬에서 실행할 수 있습니다.

```text
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
chmod 700 get_helm.sh
./get_helm.sh
```

정상적으로 설치되었는지 확인합니다.

```text
helm version --short
```

 출력 결과 예시

```text
whchoi98:~/environment $ helm version --short
v3.2.4+g0ad800e
```

### 2. 차트 Repository 구성

Stable한 저장소를 다운로드하여 아래와 같이 구성합니다.

```text
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
```

설치 한 이후에는 설치 가능한 차트를 아래와 같은 명령으로 검색할 수 있습니다.

```text
helm search repo stable
```

3. Helm 명령어 자동 완성 구성

Helm 명령에 대한 Bash 자동완성을 구성합니다.

```text
helm completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
source <(helm completion bash)
```





## Helm을 이용한 Microservice 배포.



  
  


