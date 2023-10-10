# istio 트래픽 관리 1

## 소개 및 기본 구성&#x20;

### 1.소개

Istio 서비스 메시에 마이크로서비스 기반 애플리케이션을 배포하면 서비스 모니터링 및 추적, Request(버전) 라우팅, 탄력성 테스트, 보안 및 정책 시행 등을 서비스와 애플리케이션 전반에서 일관된 방식으로 외부에서 제어할 수 있습니다.&#x20;

Istio를 사용하여 Bookinfo 버전 라우팅을 제어하려면 먼저 Subset이라고 하는 사용 가능한 버전을 정의해야 합니다.

CD(Continuous Deployment) 시나리오에서 주어진 서비스에 대해 애플리케이션 바이너리의 다른 변경을 실행하는 인스턴스의 고유한 하위 집합이 있을 수 있습니다. 이러한 변경이 반드시 다른 API 버전일 필요는 없습니다.&#x20;

다른 환경(프로덕트, 스테이징, 개발 등)에 배포된 동일한 서비스에 대한 반복적인 변경일 수 있습니다. 이것이 발생하는 일반적인 시나리오에는 A/B 테스트, 카나리아 롤아웃 등이 포함됩니다. 특정 버전의 선택은 다양한 기준(헤더, URL 등) 및/또는 각 버전에 할당된 가중치를 기반으로 결정할 수 있습니다. 각 서비스에는 모든 인스턴스로 구성된 기본 버전이 있습니다.

### 2. 기본 구성

설치한 Istio Binary에는 이미 networking을 위한 Sample App이 포함되어 있습니다.

아래를 실행합니다.&#x20;

```
kubectl -n bookinfo apply \
  -f ${HOME}/environment/istio-${ISTIO_VERSION}/samples/bookinfo/networking/destination-rule-all.yaml

```

Cloud9에서 destination-rule-all.yaml 파일을 확인해 보거나, kubectl 을 통해서 콘솔에서도 확인이 가능합니다.&#x20;

```
kubectl -n bookinfo get destinationrules -o yaml

```

### 3. 특정버전으로 라우팅

하나의 버전으로만 라우팅하기 위해 마이크로 서비스의 기본 버전을 설정하는 virtual service를 적용합니다. 이 경우 virtual service는 모든 트래픽을 마이크로 서비스의 review:v1로 라우팅합니다.&#x20;

Bookinfo 앱에는 review:v1,v2,v3 이 있습니다. 아래를 실행합니다.&#x20;

```
kubectl -n bookinfo \
  apply -f ${HOME}/environment/istio-${ISTIO_VERSION}/samples/bookinfo/networking/virtual-service-all-v1.yaml

```

Cloud9에서 virtual-service-all-v1.yaml 을 확인하거나, 아래 Kubectl 명령을 통해 확인해 봅니다.&#x20;

```
kubectl -n bookinfo get virtualservices reviews -o yaml

```

subset은 모든 review 접근에 대해 v1으로 설정되어 있습니다.&#x20;

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
```

지금 페이지를 여러 번 새로고침하고 매번 리뷰의 버전 1만 표시되는지 확인합니다.

```
echo "http://${GATEWAY_URL}/productpage"

```

웹 브라우져를 계속 해서 접근해 보거나, 아래 CURL명령을 Cloud9 창에서 실행합니다.&#x20;

![](<../../.gitbook/assets/image (412).png>)

아래 Curl을 반복 실행해 봅니다.&#x20;

```
curl XGET http://${GATEWAY_URL}/productpage -v | grep "reviews-v"

```

### 4. 사용자 ID 라우팅

특정 사용자의 모든 트래픽이 특정 서비스 버전으로 라우팅되도록 경로 구성을 변경합니다.&#x20;

이 경우 Jason이라는 사용자의 모든 트래픽은 서비스 리뷰:v2로 라우팅됩니다.아래를 실행합니다.&#x20;

```
kubectl -n bookinfo \
  apply -f ${HOME}/environment/istio-${ISTIO_VERSION}/samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml

```

Cloud9에서 virtual-service-reviews-test-v2.yaml 을 확인하거나, 아래 Kubectl 명령을 통해 확인해 봅니다.&#x20;

```
kubectl -n bookinfo get virtualservices reviews -o yaml

```

아래와 같은 내용이 포함되어 있습니다.&#x20;

```
spec:
  hosts:
    - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
```

Subset은 기본결과는 review v1로 표기되고, 로그인 사용자가 "jason"으로 로그인 하면 , review v2로 표기됩니다.&#x20;

&#x20;아래와 같이 웹 브라우져에서 로그인 해 봅니다.&#x20;

* 상단에 Sign in을 클릭합니다
* User ID : jason / Password: 없음

웹 브라우저에서는 jason으로 로그인 하면 아래와 같이 review-v2 로 로그인 됩니다.&#x20;

![](<../../.gitbook/assets/image (394).png>)

![](<../../.gitbook/assets/image (351).png>)

아래 Curl을 반복 실행해 봅니다. Curl 에서는 review-v1 이 계속 출력됩니다.&#x20;

```
curl XGET http://${GATEWAY_URL}/productpage -v | grep "reviews-v"

```

```
 $ curl XGET http://${GATEWAY_URL}/productpage -v | grep "reviews-v"
 ##중략
* Connected to a36f3d5b98fc24cb29851355830f685e-55560905.ap-northeast-2.elb.amazonaws.com (3.36.156.6) port 80 (#1)
> GET /productpage HTTP/1.1
> Host: a36f3d5b98fc24cb29851355830f685e-55560905.ap-northeast-2.elb.amazonaws.com
> User-Agent: curl/7.79.1
##중략
< server: istio-envoy
##중략
100  4294  100  4        <u>reviews-v1-55b668fc65-6np7d</u>
294    0     0   201k      0 --:--:-- --:--:-- --:--:--  209k
* Connection #1 to host a36f3d5b98fc24cb29851355830f685e-55560905.ap-northeast-2.elb.amazonaws.com left intact
```

### 5. HTTP 지연 오류 주입

복원력을 테스트하려면 사용자 jason에 대한 리뷰:v2와  ratings 마이크로서비스 사이에 7초 지연을 삽입합니다. 이 테스트는 Bookinfo 앱에 의도적으로 도입된 버그를 찾아낼 수 있습니다.&#x20;

아래를 실행합니다.&#x20;

```
kubectl -n bookinfo \
  apply -f ${HOME}/environment/istio-${ISTIO_VERSION}/samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml

```

Cloud9에서 virtual-service-ratings-test-delay.yaml을 확인하거나, 아래 Kubectl 명령을 통해 확인해 봅니다.

```
kubectl -n bookinfo get virtualservice ratings -o yaml

```

아래와 같은 내용이 포함되어 있습니다.&#x20;

```
spec:
  hosts:
  - ratings
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    fault:
      delay:
        percentage:
          value: 100.0
        fixedDelay: 7s
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
```

로그아웃한 다음 페이지의 오른쪽 상단 모서리에서 로그인을 클릭합니다. 빈 암호와 함께 jason을 사용자 이름으로 사용합니다. 지연이 표시되고 리뷰에 대한 표시 오류가 발생합니다. 다른 사람들은 오류 없이 리뷰를 볼 수 있습니다.

제품 페이지와 리뷰 서비스 사이의 제한 시간은 6초입니다.

![](<../../.gitbook/assets/image (76).png>)

![](<../../.gitbook/assets/image (415).png>)

추가적으로 복원력을 테스트하기 위해 테스트 사용자 jason에 대한 rating 마이크로 서비스에 HTTP 중단을 도입합니다. 페이지에 "Ratings service is currently unavailable"라는 메시지가 즉시 표시됩니다.

아래를 실행합니다.&#x20;

```
kubectl -n bookinfo \
  apply -f ${HOME}/environment/istio-${ISTIO_VERSION}/samples/bookinfo/networking/virtual-service-ratings-test-abort.yaml

```

Cloud9에서 virtual-service-ratings-test-abort.yaml을 확인하거나, 아래 Kubectl 명령을 통해 확인해 봅니다.

```
kubectl -n bookinfo get virtualservice ratings -o yaml

```

```
spec:
  hosts:
  - ratings
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    fault:
      abort:
        percentage:
          value: 100.0
        httpStatus: 500
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
```

Subset은 v1으로 설정되고 기록된 사용자 이름이 'jason'과 일치하는 경우 reviewer 이름 아래에 "“Ratings service is currently unavailable”라는 오류 메시지를 반환합니다.

테스트하려면 페이지 오른쪽 상단에서 로그인을 클릭하고 사용자 이름에 jason을 사용하여 빈 비밀번호로 로그인합니다. jason으로 오류 메시지가 표시됩니다.

![](<../../.gitbook/assets/image (368).png>)

![](<../../.gitbook/assets/image (367).png>)

### 6. Traffic Shaping

마이크로서비스의 한 버전에서 다른 버전으로 트래픽을 조금씩로 마이그레이션하는 방법을 보여줍니다. 이 예에서는 트래픽의 50%를 review:v1로 보내고 50%를 review:v3으로 보냅니다.

명령을 실행하여 모든 트래픽을 각 마이크로서비스의 v1 버전으로 라우팅합니다.

```
kubectl -n bookinfo \
  apply -f ${HOME}/environment/istio-${ISTIO_VERSION}/samples/bookinfo/networking/virtual-service-all-v1.yaml

```

브라우저에서 Bookinfo 사이트를 엽니다. 페이지의 리뷰 부분은 새로 고침 횟수에 상관없이 별점 없이 표시됩니다.

![](<../../.gitbook/assets/image (71).png>)

아래 명령을 통해서 트래픽의 50%를 리뷰:v1에서 리뷰:v3으로 전송할 수 있습니다.&#x20;

```
kubectl -n bookinfo \
  apply -f ${HOME}/environment/istio-${ISTIO_VERSION}/samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml

```

Cloud9에서 virtual-service-reviews-50-v3.yaml을 확인하거나, 아래 Kubectl 명령을 통해 확인해 봅니다.

```
kubectl -n bookinfo get virtualservice reviews -o yaml

```

```
spec:
  hosts:
    - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 50
    - destination:
        host: reviews
        subset: v3
      weight: 50
```

Subnet은 모든 review 요청에 대해 review-v1에 대한 트래픽의 50% 및 review-v3에 대한 트래픽의 50%로 설정됩니다.

테스트하려면 브라우저를 계속 새로고침하면 아래처럼 review-v1 및 review-v3만 표시됩니다.

![](<../../.gitbook/assets/image (176).png>)

![](<../../.gitbook/assets/image (403).png>)

reviews-v3 마이크로서비스가 안정적이라고 판단되면, 트래픽의 100%를 이 마이크로서비스로 라우팅할 수 있습니다.

```
kubectl -n bookinfo apply -f ${HOME}/environment/istio-${ISTIO_VERSION}/samples/bookinfo/networking/virtual-service-reviews-v3.yaml

```

이제 /productpage를 새로고침하면 항상 review-v3(빨간색 별 등급)이 표시됩니다.

![](<../../.gitbook/assets/image (402).png>)

