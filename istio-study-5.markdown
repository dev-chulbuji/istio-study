# Chapter 05 Traffic control : fine-grained traffic routing
- - - -

## 이 장에서는 다음과 같은 내용을 다룹니다.
* 기본적인 트래픽 라우팅
* 새로운 버전 릴리즈시 트래픽 이동
* 새로운 릴리스의 위험을 줄이기 위해 트래픽 미러링
* 클러스터 아웃 바운드 트래픽 제어

- - - -
## 배포, 릴리스

### 배포와 릴리즈의 차이점.
* CI 등을 이용하여 프로덕션 서버에 코드를 업로드 한다 - 배포
* 실제 트래픽을 업로드된 코드로 밀어 넣는다 - 릴리즈

### 배포와 릴리즈를 분리해야한다.
* 롤링 업데이트등을 이용할 경우 배포와 릴리즈가 한번에 이루어져 새로운 코드가 배포되자마자 트래픽이 유입되어 즉각적으로 장애가 발생할  수 있다.
* 배포 이후 카나리 기법등을 이용해 안정성을 검증하고, 이상이 있는 경우 이전 버전으로 빠르게 롤백할 수 있어야한다.
* Istio에서는 배포와 릴리즈를 분리하여 트래픽을 세밀하게 조정할 수 있다.

- - - -
## Routing requests with Istio
VirtualService Istio 리소스를 이용한 트래픽 라우팅 방식을 자세히 살펴 보기.

### Dark launch
Request header를 보고 판단하여 라우팅 제어를 할 경우 Dark launch라는 방식으로 배포를 할 수 있습니다.
상당수의 사용자는 안정적인 버전으로 보내지고, 특정 클래스의 사용자는 최신 버전으로 보내져, 다른 사람들에게 영향을 끼치지 않으며 새로운 기능을 특정 그룹에만 노출 시킬 수 있습니다.

#### how to?
catalog service Version 1 모든 트래픽은 v1으로 보내진다.
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog-vs-from-gw
spec:
  hosts:
  - "catalog.istioinaction.io"
  gateways:
  - catalog-gateway
  http:
  - route:
    - destination:
        host: catalog
```

새로 만들 DestinationRule에서 label을 이용해 라우팅이 가능한 룰을 생성한다.
```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: catalog
spec:
  host: catalog
  subsets:
  - name: version-v1
    labels:
      version: v1
  - name: version-v2
    labels:
		version: v2
```

catalog service Version 1 VertualService를 업데이트한다.
subset을 추가하여 label 기반 라우팅을 적용.
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog-vs-from-gw
spec:
  hosts:
  - "catalog.istioinaction.io"
  gateways:
  - catalog-gateway
  http:
  - route:
    - destination:
        host: catalog
        subset: version-v1 <<<<
```

현재 구조는 아래와 같다.
![](chap05/page134image19435792.png)

catalog V2 deploymentes를 생성한 후.
catalog service VertualService를 replace 해준다.
특정 헤더를 가진 요청만이 V2로 보내지게 되는것이다.
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog-vs-from-gw
spec:
  hosts:
  - "catalog.istioinaction.io"
  gateways:
  - catalog-gateway
  http:
  - match:
      - headers:
          x-istio-cohort:
            exact: "internal"
      route:
      - destination:
          host: catalog
          subset: version-v2
  - route:
    - destination:
        host: catalog
        subset: version-v1
```

최종 적용된 그림은 아래와 같다.
이러한 istio에서의 라우팅은 Envoy에서 파생된것이며, 이 외에도 LoadBalancing, Connection pool등의 기능 역시 사용 가능하다.
![](chap05/page135image19690496.png)

## Traffic shifting
카나리아 혹은 점진적인 배포에 대해 알아보자.
모든 실시간 트래픽을 Weight 기분으로 특정 서비스 세트에 배포하는 방식을 보여줍니다.

예를 들어, 서비스의 v2를 내부 직원에게만 Dark launch로 시작한 후, 모두에게 천천히 릴리즈 하려면 모든 트래픽 중 10% 만을 v2로 라우팅 시킬 수 있습니다.
v2 전체 트래픽중 부정적인 영향이 얼마인지 체크하며 제어하여 릴리즈 위험을 줄일 수 있습니다.
문제가 있는 경우 릴리즈를 롤백하는데, Weight를 0%로 조정하는것으로 간단하게 롤백 할 수 있습니다.

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog
spec:
  hosts:
  - catalog
  gateways:
- mesh http:
  - route:
    - destination:
        host: catalog
        subset: version-v1
      weight: 90 <<<<
    - destination:
        host: catalog
        subset: version-v2
      weight: 10 <<<<
```

* Weight 값의 합은 100과 동일해야하며, 동일하지 않을 경우 예기치 않은 트래픽 라우팅이 발생할 수 있습니다.
* 이상적인 점진적 트래픽 이동을 위해서는 CI/CD나 배포 파이프라인을 이용해 자동화를 해줍시다.

### 주의점
* 이 작업을 수행 할 때 여러 버전을 동시에 실행할 수 있도록 서비스를 구축해야합니다.
* [Traffic Shadowing With Istio: Reducing the Risk of Code Release - DZone Microservices](https://dzone.com/articles/traffic-shadowing-with-istio-reducing-the-risk-of)
* [Advanced Traffic-Shadowing Patterns for Microservices With Istio Service Mesh - DZone Microservices](https://dzone.com/articles/advanced-traffic-shadowing-patterns-for-microservi)


## Lowering risk even further: Traffic mirroring
request-level routing 과 traffic shifting을 사용하여 릴리즈 위험을 줄일 수 있습니다.
두 기술 모두 실시간 트래픽/요청을 사용하며 부정적인 영향을 미칠 수있는 범위를 제어하지만, 실제 사용자에게 영향을 줄 수 있습니다.

여기에서 또 다른 방법은 트래픽을 미러링하여 프로덕션 트래픽의 복사본을 만들어 새로운 deployments로 보내는 것입니다.
미러링 방식을 이용하여 실제 사용자에게 영향을주지 않고 새로운 코드가 어떻게 작동하는지 확인해 볼 수 있습니다.

![](chap05/page141image19469808.png)

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catalog
spec:
  hosts:
  - catalog
  gateways:
- mesh http:
  - route:
    - destination:
        host: catalog
        subset: version-v1
      weight: 100
    mirror:
      host: catalog
      subset: version-v2
```

위와 같은 설정을 적용하면 라이브 트래픽의 100%를 v1으로 라우팅하지만 v2로 미러링하게됨.
미러링을 수행하는 istio proxy는 요청의 실패를 무시하기 때문에 v1 요청에 영향을 줄 수 없다.

미러링된 요청의 호스트에는 -shadow 라는 suffix가 붙게된다.
어플리케이션에서는 이를 고려하여 미러링 요청에 대한 처리를 구분할 수 있다.
ex) 트랜잭션을 롤백하지 않거나, 시도하지 않는다.

## Routing to services outside your cluster by using Istio’s service discovery
기본적으로 Istio는 서비스 메시 외부의 모든 아웃바운드 트래픽을 허용합니다.
예를 들어, 메시 외부 웹 사이트나 서비스와 통신이 허용됩니다.

모든 트래픽은 먼저 서비스 메시 사이드카 프록시 (Istio proxy) 를 통과하기 때문에
Istio의 기본 정책을 변경하여 모든 아웃바운드 트래픽을 거부 할 수 있습니다.

메시 내에서 모든 트래픽을 차단하는 것은 메시 내 서비스 또는 어플리케이션이 손상 될 경우 나쁜 사람들의 "phoning home"을 방지하기위한 기본적인 심층 방어 자세입니다.

그러나 Istio를 사용하여 외부 트래픽을 차단하는 것만으로는 충분하지 않습니다.
손상된 pod는 프록시를 우회 할 수 있습니다. 따라서 방화벽과 같은 추가 트래픽 차단 메커니즘을 갖춘 심층 방어 접근 방식이 필요합니다.

예를 들어, 침입자가 취약점으로 인해 특정 서비스를 제어 할 수있는 경우, 코드를 삽입하거나 서비스를 조작하여 자신이 제어하는 ​​서버에 도달 할 수 있습니다.
침입자가 손상된 서비스를 제어 할 수 있다면 회사에 민감한 데이터 및 지적 재산을 유출할수 있습니다.

### 모든 아웃바운드 트래픽 차단하기.
![](chap05/page144image56232384.png)
아래 커맨드를 입력하면 Istio의 기본값을 "ALLOW_ANY"에서 "REGISTRY_ONLY"로 변경할 수 있습니다.
즉, 서비스 메시 레지스트리에 명시적으로 허용된 경우에만 아웃바운드를 허용합니다.
이 설정을 적용하기 위해서는 모든 pod을 삭제하여야 합니다.
```
$ kubectl get configmap istio -n istio-system -o yaml \
| sed 's/mode: ALLOW_ANY/mode: REGISTRY_ONLY/g' \
| kubectl replace -n istio-system -f -
..
..
..
$ kubectl delete pod --all -n istioinaction
```

### ServiceEntry
위 설정을 적용할 경우 메시 내부의 서비스가 메시 외부의 서비스와 통신 할 수있는 방법이 필요합니다.

istio pilot에서 kube registry와 service entry를 통합하여 관리할 수 있습니다.
![](chap05/page145image56595952.png)

* Istio는 메시 내에서 액세스 할 수있는 모든 서비스의 내부 서비스 레지스트리를 구축합니다.
* 레지스트리를 메시 내의 서비스가 다른 서비스를 찾는 데 사용할 수있는 service-discovery 레지스트리의 표준 표현이라고 볼 수 있다.
* 외부의 서비스와 통신하려면 service-discovery 레지스트리에 등록해야합니다.
* ServiceEntry는 서비스 레지스트리에 항목을 삽입하는 데 사용할 수있는 레지스트리 메타 데이터를 캡슐화합니다.

```
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: jsonplaceholder
spec:
  hosts:
  - jsonplaceholder.typicode.com
  ports:
  - number: 80
name: http
    protocol: HTTP
  resolution: DNS
  location: MESH_EXTERNAL
```

forum 서비스를 설치하며, sidecar를 주입해준다.
설치 후에 당연하게도 jsonplaceholder.typicode.com 호출이 불가능한 상태이다.
```
$ kubectl apply -f <(istioctl kube-inject -f services/forum/kubernetes/forum-all.yaml)
```

service entry를 적용한 후에는 jsonplaceholder.typicode.com 호출이 가능해진다.
```
$  kubectl apply -f chapters/chapter5/forum-serviceentry.yaml
```

* 이 ServiceEntry 리소스는 Istio의 서비스 레지스트리에 항목을 삽입하여 호스트 jsonplaceholder.typicode.com을 호출 할 수 있도록합니다.
* jsonplaceholder.typicode.com는 샘플 REST API를 제공하는 서비스입니다..
* jsonplaceholder.typicode.com을 활용하는 샘플 "forum" 애플리케이션을 설치하여 통신이 가능한지 확인합니다.
![](chap05/page147image56133456.png)
위 그림과 같이 from 서비스에서 jsonplaceholder.typicode.com 요청이 가능해집니다.
