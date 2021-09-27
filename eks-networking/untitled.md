---
description: 'Update : 2021-09-27'
---

# Multus

## Multus 소개 

[Multus](https://github.com/Intel-Corp/multus-cni)는 쿠버네티스의 CRD 기반 네트워크 오브젝트를 사용하여 쿠버네티스에서 멀티 네트워킹 기능을 지원하는 멀티 CNI 플러그인입니다.

Multus는 CNI 명세를 구현하는 모든 [레퍼런스 플러그인](https://github.com/containernetworking/plugins) 및 3rd Party 플러그인을 지원합니. 또한, Multus는 쿠버네티스의 클라우드 네이티브 애플리케이션과 NFV 기반 애플리케이션을 통해 쿠버네티스의 [SRIOV](https://github.com/hustcat/sriov-cni), [DPDK](https://github.com/Intel-Corp/sriov-cni), [OVS-DPDK 및 VPP](https://github.com/intel/vhost-user-net-plugin) 워크로드를 지원합니다.

Multus는 Pod에 멀 네트워크 인터페이스를 첨부할 수 있는 Kubernetes용 오픈 소스 CNI 플러그인입니다. Multus는 메타 플러그인을 기반으로 Pod에 첨부된 다중 네트워크 인터페이스를 작동하는 추가 CNI 플러그인을 지원합니다. 다중 인터페이스를 포함한 Pod가 일반적으로 필요한 사용 사례에는 Kubernetes에 대한 5G 및 스트리밍 네크워크 등이 있습니다. EKS가 지원하는 Multus를 사용하여 이러한 환경에 걸쳐 어드밴스드 네트워킹을 활성화함으로써 사용자에게 고품질 콘텐츠를 전달하는 컨테이너화 네트워크 기능을 실행할 수 있습니다.

랩에서는 아래와 같이 US-WEST-2 \(오레곤\) 리전에서 EKS 기반으로  Multus를 적용하는 방안을 소개 합니다. 





  


