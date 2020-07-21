# Loadbalancer 기반 배포

## Loadbalancer 서비스 타입 소개.

앞서 Service 배포의 ClusterIP 타입을 Kubernetes Dashboard 배포를 통해 확인했습니다.

그외에 NodePort 타입으로도 구성이 가능합니다. 아래는 NodePort 타입으로 구성하는 아키텍쳐입니다. 

![](../.gitbook/assets/image%20%286%29.png)

노드 타입 구조의 경우 노드의 IP주소와 Port에 종속되고, 확장성과 유연함에 한계가 있습니다.

대부분은 Loadbalancer 타입과 Ingress 기반과 Service 가 연계되어 확장성과 유연함을 갖는 구조를 제공합니다.

![](../.gitbook/assets/image%20%283%29.png)

## Loadbalancer 서비스 기반 구성

1.배포용 yaml 복제

LAB에서 사용할 App을 복제합니다.

```text
cd ~/environment
git clone https://github.com/brentley/ecsdemo-frontend.git
git clone https://github.com/brentley/ecsdemo-nodejs.git
git clone https://github.com/brentley/ecsdemo-crystal.git
```

정상적으로 복제 이후 Cloud9에서 아래와 같이 확인됩니다.

![](../.gitbook/assets/image%20%2814%29.png)

2. 



