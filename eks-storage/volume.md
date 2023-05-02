# 볼륨

## 소개



컨테이너 내의 디스크에 있는 파일은 임시적이며, 컨테이너에서 실행될 때 애플리케이션에 적지 않은 몇 가지 문제가 발생합니다. 한 가지 문제는 컨테이너가 크래시될 때 파일이 손실된다는 것입니다. kubelet은 컨테이너를 다시 시작하지만 초기화된 상태입니다. 두 번째 문제는 `Pod`에서 같이 실행되는 컨테이너간에 파일을 공유할 때 발생합니다. 쿠버네티스 [볼륨](https://kubernetes.io/ko/docs/concepts/storage/volumes/) 추상화는 이러한 문제를 모두 해결합니다.



## 임시볼륨 (empty volume)

파드가 시작될 때 빈 상태로 시작되며, 저장소는 로컬의 kubelet 베이스 디렉터리(보통 루트 디스크) 또는 램에서 제공됩니다.

아래와 같이 새로운 임시볼륨을 생성해 봅니다.

```
cat << EOF > ~/environment/empty_stg.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: empty
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-volume-1
  namespace: empty
spec:
  containers:
  - name: container1
    image: whchoi98/network-multitool
    volumeMounts:
    - name: empty-dir
      mountPath: /mount1
  - name: container2
    image: kubetm/init
    volumeMounts:
    - name: empty-dir
      mountPath: /mount2
  volumes:
  - name : empty-dir
    emptyDir: {}
EOF

```

생성된 yml 파일을 아래 명령어로 실행합니다.

```
kubectl apply -f  ~/environment/empty_stg.yaml
kubectl -n empty get pods

```

생성된 첫번째 컨테이너 (Container 1)의 Shell에 접속해서 아래 명령을 수행합니다. \
(앞서 설치한 K9를 활용해서 접속할 수 있습니다.)

```
echo "This is empty-volumes" >> /mount1/empty.txt
cat /mount1/empty.txt

```

생성된 두번째 컨테이너 (Container2)의 Shell에 접속해서 Container1 에서 생성한 텍스트 값이 있는지 확인해 봅니다.

```
cat /mount2/empty.txt

```

임시 볼륨은 Pod내에서만 존재하기 때문에 Pod가 삭제되면 더 이상 데이터는 보존되지 않습니다.



## hostPath

`hostPath` 볼륨은 호스트 노드의 파일시스템에 있는 파일이나 디렉터리를 파드에 마운트하는 방식입니다. hostPath는 호스트 내의 Pod들이 볼륨을 공유하는 방식이기 때문에 Worker Node가 삭제되면 데이터도 삭제됩니다.



이미 생성된 Node들 중에서 1개의 노드를 선택해서 새로운 라벨링을 부여합니다.

아래는 구성 예제입니다.

```
kubectl get nodes -l nodegroup-type=managed-frontend-workloads
kubectl label nodes ip-10-11-1-159.ap-northeast-2.compute.internal hostpath=true
kubectl get nodes -l hostpath=true

```

새로운 네임스페이스와 Pod 1개를 구성합니다.

```
cat << EOF > ~/environment/hostpath_stg1.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: hostpath
---
apiVersion: v1
kind: Pod
metadata:
  name: hostpath1
  namespace: hostpath
spec:
  containers:
  - name: container1
    image: whchoi98/network-multitool
    volumeMounts:
    - name: hostpath
      mountPath: /hostpath-mount
  volumes:
  - name: hostpath
    hostPath:
      path: /tmp
      type: Directory
  nodeSelector:
    hostpath: "true"
EOF
kubectl apply -f  ~/environment/hostpath_stg1.yaml
kubectl -n hostpath get pods -o wide

```

&#x20;container1에 shell로 접근해서 새로운 Text 파일을 만들어 봅니다.

```
echo "This is hostpath-mount-01" >> /hostpath-mount/hostpath.txt

```

하나의 pod를 추가로 생성해 봅니다.

```
cat << EOF > ~/environment/hostpath_stg2.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: hostpath
---
apiVersion: v1
kind: Pod
metadata:
  name: hostpath2
  namespace: hostpath
spec:
  containers:
  - name: container2
    image: whchoi98/network-multitool
    volumeMounts:
    - name: hostpath
      mountPath: /hostpath-mount
  volumes:
  - name: hostpath
    hostPath:
      path: /tmp
      type: Directory
  nodeSelector:
    hostpath: "true"
EOF
kubectl apply -f  ~/environment/hostpath_stg2.yaml
kubectl -n hostpath get pods -o wide


```

새롭게 생성된 Container2에 Shell로 접근해서 아래 명령을 수행해 봅니다.

```
echo "This is hostpath-mount-02" >> /hostpath-mount/hostpath.txt

```

이제 앞서 **`hostpath=true`** 라벨링을 부여한 노드로 접속해서, 실제 노드 값을 확인해 봅니다.

Session Manager를 통해서 접속이 가능합니다.

<figure><img src="../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (1) (5).png" alt=""><figcaption></figcaption></figure>

```
cat /tmp/hostpath.txt

```

노드에서 실행한 값이 앞서 Container1,2에서 실행한 값을 모두 포함하는지 확인해 봅니다.

노드기반의 hostPath는 노드가 삭제되면 모든 데이터는 소멸됩니다.



## 퍼시스턴트 볼륨

퍼시스턴트볼륨은 사용자 및 관리자에게 스토리지 사용 방법에서부터 스토리지가 제공되는 방법에 대한 세부 사항을 추상화하는 API를 제공합니다. 이를 위해 퍼시스턴트볼륨(PV) 및 퍼시스턴트볼륨클레임(PVC)이라는 두 가지 새로운 API 리소스를 제공합니다.

_퍼시스턴트볼륨_ (PV)은 관리자가 프로비저닝하거나 [스토리지 클래스](https://kubernetes.io/ko/docs/concepts/storage/storage-classes/)를 사용하여 동적으로 프로비저닝한 클러스터의 스토리지입니다. PV는 Volumes와 같은 볼륨 플러그인이지만, PV를 사용하는 개별 파드와는 별개의 라이프사이클을 가진다. 이 API 오브젝트는 NFS, iSCSI 또는 클라우드 공급자별 스토리지 시스템 등 스토리지 구현에 대한 세부 정보를 포함하고 있습니다.

_퍼시스턴트볼륨클레임_ (PVC)은 사용자의 스토리지에 대한 요청입니다. 이것은 파드와 비슷하다. 파드는 노드 리소스를 사용하고 PVC는 PV 리소스를 사용한다. 파드는 특정 수준의 리소스(CPU 및 메모리)를 요청할 수 있지만, 클레임은 특정 디스크 크기 및 접근 모드를 요청할 수 있습니다. (예: ReadWriteOnce, ReadOnlyMany 또는 ReadWriteMany로 마운트 할 수 있음. [AccessModes](https://kubernetes.io/ko/docs/concepts/storage/persistent-volumes/#%EC%A0%91%EA%B7%BC-%EB%AA%A8%EB%93%9C) 참고).

퍼시스턴트볼륨클레임을 사용하면 사용자가 추상화된 스토리지 리소스를 사용할 수 있지만, 다른 문제들 때문에 성능과 같은 다양한 속성을 가진 퍼시스턴트볼륨이 필요한 경우가 일반적입니다. 클러스터 관리자는 사용자에게 해당 볼륨의 구현 방법에 대한 세부 정보를 제공하지 않고 크기와 접근 모드와는 다른 방식으로 다양한 퍼시스턴트볼륨을 제공할 수 있어야 하며, 이러한 요구를 위해서는 _스토리지클래스_ 리소스가 있습니다.







