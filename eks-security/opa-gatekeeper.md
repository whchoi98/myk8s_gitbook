---
description: 'Update : 2023-09-21'
---

# OPA Gatekeeper

## OPA 소개와 Gatekeeper 설치

### 1.OPA 소개

OPA는 K8S 뿐만 아니라 다양한 영역에서 운영환경에 대한 정책을 적용하고 통합하기 위해 사용되는 Open Source 기반의 플랫폼입니다. 현재 CNCF 의 Graduated Project 상태로 정책의 코드화된 관리를 목적으로 다양하게 사용되고 있습니다. AWS 에서는 EKS 뿐만 아니라 AWS Config 서비스에서도 OPA 기반의 Config 규칙을 사용하실 수 있습니다.

{% hint style="info" %}
OPA 는 Rego 라는 별도의 Language 를 통해 정책을 생성/관리합니다. Rego Language는 [OPA Playground ](https://play.openpolicyagent.org/)에서 작성한 OPA 정책을 확인할 수 있습니다.
{% endhint %}



### 2.OPA Gatekeeper 소개

OPA Gatekeeper 는 OPA 에서 생성된 정책을 K8S 환경에서 적용하기 위해 사용하는 K8S 를 위한 Adminssion Webhook 입니다. OPA Gatekeep 의 기본적인 Flow 는 아래와 같습니다.

<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

### 3.OPA Gatekeeper 설치

아래의 명령을 이용하여 Prebuilt Image 기반으로 OPA Gatekeeper 를 설치합니다.

```
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.8/deploy/gatekeeper.yaml

```

정상적으로 설치되었는지 확인합니다.

```
kubectl get pods -n gatekeeper-system

```

실행결과 예제는 아래와 같습니다.

```
$ kubectl get pods -n gatekeeper-system
NAME                                             READY   STATUS    RESTARTS   AGE
gatekeeper-audit-5dc6b7649c-z4kb4                1/1     Running   0          51s
gatekeeper-controller-manager-6d7579bf5c-8l7qx   1/1     Running   0          50s
gatekeeper-controller-manager-6d7579bf5c-mmjwc   1/1     Running   0          51s
gatekeeper-controller-manager-6d7579bf5c-wnwms   1/1     Running   0          50s
```

OPA Gatekeepr 설치와 관련한 자세한 사항은 [설치 가이드 ](https://open-policy-agent.github.io/gatekeeper/website/docs/install/)를 참고하실 수 있습니다.

아래의 명령을 실행하여 "audit-controller" 와 "controller-manager" 의 Log 를 확인합니다.

```
# audit-controller log 확인
kubectl logs -l control-plane=audit-controller -n gatekeeper-system

```

<pre><code># gatekeeper-system log 확인
kubectl logs -l control-plane=controller-manager -n gatekeeper-system
<strong>
</strong></code></pre>

## Container Image 제한하기

### 1.Container Image 사용제한하기

1. 등록된 Container Image 만을 허용
2. Priviledge 권한이 부여된 Container 실행 차단
3. 등록된 Repository 에서만 Container Image 사용을 허용

K8S 에 OPA Gatekeeper 의 정책을 적용하기 위해서는 "ConstraintTemplate" 과 "Constraint" 를 함께 구성해야 합니다.&#x20;

* "ConstraintTemplate" 는 **정책의 전체적인 내용(무엇을 검사하고 어떤 경고 메시지를 보낼지 등)을 정의**
* "Constraint" 는 "ConstraintTemplate" 에 정의된 **정책의 대상을 지정**하는 용도입니다.

Container Image 사용 제한을 위한 "ConstraintTemplate" 을 생성합니다.

아래의 내용을 실행하여 새로운 "ConstraintTemplate" 을 생성합니다.

```
mkdir ~/environment/opa
cat  << EOF > ~/environment/opa/constraint-template-image.yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: enforceimagelist
spec:
  crd:
    spec:
      names:
        kind: enforceimagelist
      validation:
        openAPIV3Schema:
          properties:
            images:
              type: array
              items: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package enforceimagelist

        allowlisted_images = {images |
            images = input.parameters.images[_]
        }
    
        images_allowlisted(str, patterns) {
            image_matches(str, patterns[_])
        }
    
        image_matches(str, pattern) {
            contains(str, pattern)
        }

        violation[{"msg": msg}] {
          input.review.object
          image := input.review.object.spec.containers[_].image
          name := input.review.object.metadata.name
          not images_allowlisted(image, allowlisted_images)
          msg := sprintf("pod %q has invalid image %q. Please, contact Security Team. Follow the allowlisted images %v", [name, image, allowlisted_images])
        }
EOF

```



정책 적용의 대상을 지정하는 "Constraint" 를 생성합니다.

constraint-image.yaml 파일에는 정책에서 허용하고자하는 Container Image 의 리스트가 정의되어 있습니다. 정책이 적용되면 여기에 지정되어 있지 않은 Container Image 의 사용은 제한되게 됩니다.

<pre><code><strong>cat  &#x3C;&#x3C; EOF > ~/environment/opa/constraint-image.yaml 
</strong>apiVersion: constraints.gatekeeper.sh/v1beta1
kind: enforceimagelist
metadata:
  name: k8senforceallowlistedimages
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    images:
      - $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/eks-security-shared
      - amazon/aws-node-termination-handler
      - amazon/aws-alb-ingress-controller
      - amazon/aws-efs-csi-driver
      - amazon/cloudwatch-agent
      - docker.io/amazon/aws-alb-ingress-controller
      - grafana/grafana
      - prom/alertmanager
      - prom/prometheus
      - openpolicyagent/gatekeeper
      - amazon/aws-cli
      - busybox
      - nginx
      - falco
EOF

</code></pre>

아래의 명령을 실행하여 "ConstraintTemplate" 와 "Constraint" 를 적용합니다.

```
kubectl apply -f ~/environment/opa/constraint-template-image.yaml 
kubectl apply -f ~/environment/opa/constraint-image.yaml

```

생성된 "constraintTemplate"과 "constranit" 를 확인합니다.

```
kubectl get constrainttemplate
kubectl get constraint


```

아래 명령을 통해서 "constraintTemplate"에 적용된 정책을 확인해 봅니다.

```
# constraintTemplate 상세 내용 확인
kubectl get constrainttemplate -o yaml enforceimagelist
```

constrainttemplate 내에 Container Image 를 선택적으로 차단하는 내용이 포함되어 있는 것을 확인할 수 있습니다.

```
생략
targets:
  - rego: |
      package enforceimagelist

      allowlisted_images = {images |
          images = input.parameters.images[_]
      }

      images_allowlisted(str, patterns) {
          image_matches(str, patterns[_])
      }

      image_matches(str, pattern) {
          contains(str, pattern)
      }

      violation[{"msg": msg}] {
        input.review.object
        image := input.review.object.spec.containers[_].image
        name := input.review.object.metadata.name
        not images_allowlisted(image, allowlisted_images)
        msg := sprintf("pod %q has invalid image %q. Please, contact Security Team. Follow the allowlisted images %v", [name, image, allowlisted_images])
      }
    target: admission.k8s.gatekeeper.sh
이하 생략
```

"Constraint" 의 상세 내용을 확인합니다.

```
kubectl get constraint -o yaml k8senforceallowlistedimages

```

아래와 같이 지정된 Container Image 리스트가 포함되어 있는 것을 확인할 수 있습니다.

```
  parameters:
    images:
    - 300861432382.dkr.ecr.ap-northeast-2.amazonaws.com/eks-security-shared
    - amazon/aws-node-termination-handler
    - amazon/aws-alb-ingress-controller
    - amazon/aws-efs-csi-driver
    - amazon/cloudwatch-agent
    - docker.io/amazon/aws-alb-ingress-controller
    - grafana/grafana
    - prom/alertmanager
    - prom/prometheus
    - openpolicyagent/gatekeeper
    - amazon/aws-cli
    - busybox
    - nginx
    - falco
```

### 2.Container Image 사용제한 확인

Container Image 사용제한을 위한 OPA Gatekeeper 정책이 모두 적용된 상태입니다. 이제 적용된 정책을 Pod 생성을 통해 검증해 봅니다.

"Constraint" 에 등록되지 않은 docker image 를 사용하는 Pod 생성을 위한 YAML 파일을 생성합니다.

```
cat  << EOF > ~/environment/opa/pod-with-invalid-image.yaml
apiVersion: v1
kind: Pod
metadata:
  name: invalid-image
  labels:
    app: invalid-image
  namespace: default
spec:
  containers:
  - image: docker:latest
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    name: invalid-docker
  restartPolicy: Always
EOF

```

OPA 정책에 위반된 Pod를 배포합니다.

```
kubectl apply -f ~/environment/opa/pod-with-invalid-image.yaml

```

아래와 같이 허용되지 않은 Container Image 를 사용한 Pod 의 배포는 Error 와 함께 차단되는 것을 확인할 수 있습니다.

```
$ kubectl apply -f ~/environment/opa/pod-with-invalid-image.yaml
Error from server (Forbidden): error when creating "/home/ec2-user/environment/opa/pod-with-invalid-image.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [k8senforceallowlistedimages] pod "invalid-image" has invalid image "docker:latest". Please, contact Security Team. Follow the allowlisted images {"300861432382.dkr.ecr.ap-northeast-2.amazonaws.com/eks-security-shared", "amazon/aws-alb-ingress-controller", "amazon/aws-cli", "amazon/aws-efs-csi-driver", "amazon/aws-node-termination-handler", "amazon/cloudwatch-agent", "busybox", "docker.io/amazon/aws-alb-ingress-controller", "falco", "grafana/grafana", "nginx", "openpolicyagent/gatekeeper", "prom/alertmanager", "prom/prometheus"}
```

아래와 같이 허용된 Conatiner Image를 사용하는, 즉 "Constraint" 에 등록되어 있는 "busybox" 를 사용하는 Pod 을 배포해 봅니다.

```
cat  << EOF > ~/environment/opa/pod-with-valid-image.yaml
apiVersion: v1
kind: Pod
metadata:
  name: valid-image
  labels:
    app: valid-image
  namespace: default
spec:
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    name: valid-busybox
  restartPolicy: Always
EOF

```

OPA 정책에 위반되지 않는 Pod를 배포합니다.

```
kubectl apply -f ~/environment/opa/pod-with-valid-image.yaml

```

정상적으로 배포되는 것을 확인 할 수 있습니다.

3.Container Image 사용제한 정책 삭제

LAB 진행을 위해 아래와 같이 생성된 정책을 삭제 합니다.

```
kubectl delete -f ~/environment/opa/pod-with-valid-image.yaml
kubectl delete -f ~/environment/opa/constraint-image.yaml
kubectl delete -f ~/environment/opa/constraint-template-image.yaml 

```
