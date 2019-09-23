# Istio Study #2.2 Isio Control Plane 알아보기

이 문서는 [Istio in Action v5 MEAP](https://www.manning.com/books/istio-in-action) 
내용을 기반으로 작성했습니다. 

[2.1 Kubernetes에 Istio 배포하기](/blog/istio/2019/09/21/istio-study-2.1.html)
에서 Istio의 control plane의 모든 구성 요소를 설치했습니다. 

Control plane은 사용자에게 아래와 같이 service mesh를 제어, 관찰, 관리, 설정 할 수 있는 방법을 제공합니다.

- 라우팅, 복원
- Data plane에 로컬화된 설정을 제공하기 위한 API 제공
- Service discovery 
- 할당량을 설정 및 사용량을 제한
- 사용 정책 설정 
- 인증서 발급 및 교체 
- Workload 식별
- telemetry(원격 측정) 통합 수집
- 네트워크 접근 제어

이러한 기능은 control plane의 각 구성 요소에 분산되어 있습니다.
이어서 핵심 구성 요소에 대해 더 살펴보겠습니다.

## 1.Istio Pilot

Istio Pilot은 사용자와 운영자가 service proxy를 data plane에 구성할 수 있게 해줍니다.

Pilot은 플랫폼에 독립적이며 플랫폼 adapter를 사용해 플랫폼에 특화된 서비스를 사용합니다.
예를 들어, Kubernetes 플랫폼에서는 Kubernetes에서 제공하는 방식으로 service registration 및 discovery 서비스를 활용합니다.

![](./images/istio-study-2.2-001.png)
[출처: Istio in Action MEAP Edition]

Pilot의 설정을 통해 traffic 유입, 라우팅, 서비스 복원 등을 관리할 수 있습니다.
예를 들어, 요청 정보에 따라 어떤 버전의 서비스를 사용하는지 또는 배포를 할 때 버전간의 트래픽을 어떻게 이동하는지 등의 traffic을 관리할 수 있습니다.
또한 요청에 대한 timeout, retries, circuit break와 같은 서비스 복원을 관리할 수 있습니다.

Istio는 기본적으로 Envoy를 service proxy로 사용합니다.
아래 예제에서 Pilot의 설정을 살펴보겠습니다.

한 서비스가 catalog 서비스와 통신하려고 합니다.
traffic header 정보에 `x-dark-launch`의 갑싱 `v2`이면 catalog 서비스의 `v2`로 라우팅을 하려합니다.
이 경우 Pilot의 설정을 아래와 같이 설정할 수 있습니다.

![](./images/istio-study-2.2-002.png)
[출처: Istio in Action MEAP Edition]

1. 매칭하려는 요청 조건
2. 매칭하려는 요청 조건 상세
3. 매칭된 경우 라우팅 목적지
4. 매칭되지 않은 다른 경우 라우팅 목적지

이 설정을 Istio 환경에 적용하려면 아래와 같이 `Kubectl` tool을 활용하면됩니다.
(catalog-service.yaml 파일에는 위에서 예로 든 설정이 저장되어 있습니다.)

```
$ kubectl create -f catalog-service.yaml
```

이 부분에 대한 좀 더 상세한 내용은 추후 살펴보기로 하겠습니다.

<p class="tip-title">참고</p>
<p class="tip-content">
Istio의 VirtualService등의 리소스는 Kubernetes의 CRD(Custom
Resource Definitions)를 통해 구현되었습니다.
</p>

Istio는 `VirtualService`와 같은 Istio 특화된 설정을 Envoy의 native 설정으로 변환합니다.
예를 들어, Istio Pilot은 위의 설정을 아래와 같은 data plane API 형태로 service proxy에 노출합니다.

```
"domains":[
   "catalog.prod.svc.cluster.local"
],
"name":"catalog.prod.svc.cluster.local:80",
"routes":[
   {
      "match":{
         "headers":[
            {
               "name":"x-dark-lauch",
               "value":"v2"
            }
         ],
         "prefix":"/"
      },
      "route":{
         "cluster":"outbound|80|v2|catalog.prod.svc.cluster.local",
         "use_websocket":false
      }
   },
   {
      "match":{
         "prefix":"/"
      },
      "route":{
         "cluster":"outbound|80|v1|catalog.prod.svc.cluster.local",
         "use_websocket":false
      }
   }
]
```

<p class="tip-title">참고</p>
<p class="tip-content">
위 data plane API는 Istio Pilot(Envoy's discovery APIs로 구현)에 의해 노출됩니다.
이 API를 통해 data plane은 중지나 재시작없이 구성을 동적으로 반영할 수 있습니다.
이 API는 *xDS*라고 하는데 추후 Envoy proxy를 다룰때 좀 더 상세히 살펴보겠습니다.
</p>

## 2.Ingress 및 Egress gateway

Application 및 service는 cluster 외부의 application과 통신해야하는 경우가 있습니다.
예를 들어, 모놀리틱 앱, 상용 소프트웨어, 메세징 큐, 데이터베이스, 3rd pary 앱 등이 있습니다. 
운영자는 이런 traffic이 cluster로 들어오거나 나가는 것을 적절하게 구성해야합니다.

![](./images/istio-study-2.2-003.png)
[출처: Istio in Action MEAP Edition]

Istio에서 이런 기능을 제공하는 구성 요소는 istio-ingressgateway, istio-egressgateway라고 합니다.
이 두 가지 구성 요소는 Istio 설정을 이해할 수 있는 Envoy proxy 그 자체입니다. 

이것들은 기술적으로 control plane의 부분이 아니지만 service mesh에서 중요한 역할을 합니다.
data plane의 service proxy와 유사하게 구성되어 있으며
차이점은 application과 독립적이며 단순히 traffic이 들어오고 나가는 것만을 관리합니다.

이 구성 요소에 대한 자세한 내용은 추후에 좀 더 살펴보겠습니다.

## 3.Istio Citadel

Istio service mesh에서 service proxy는 application 인스턴스와 함께 실행됩니다.
하나의 application이 다른 application을 호출할 때 송신 및 수신하는 application의 proxy들은 직접 통신합니다.

Istio의 주요 특징 중 하나는 두 service간의 통신을 암호화 한다는 것입니다.
X.509 인증서를 사용해 traffic을 암호화합니다.
사실, 이 인증서에는 [SPIFFE(Secure Production Identity Framework For Everyone)](https://spiffe.io/)에 의해 Workload의 식별자가 삽입됩니다.
이것은 application이 어떤 인증서를 가지고 있지 않아도 강력한 mTLS(mutual Transport Layer Security)를 제공합니다.

Istio Citadel은 위와 같은 보안을 다루는 구성 요소로 인증서의 증명, 발급, 마운트 및 교체를 처리합니다. 
이에 대한 상세한 내용은 Istio의 보안 부분을 다룰 때 좀 더 상세히 살펴보겠습니다.

![](./images/istio-study-2.2-004.png)
[출처: Istio in Action MEAP Edition]

## 4.Istio Mixer

Istio의 Mixer는 service mesh의 주요 세 가지 기능을 다루는 control plane 구성 요소입니다.

1. telemetry(원격 측정) 수집
2. traffic, 인증 정책 정의 및 실행
3. 할당량 관리, 사용량 제한

Application은 메트릭, 로그, 추적 데이터 등의 정보를 수집하기 위한 백엔드 인프라 서비스와 통신해야하는 경우가 있습니다.
이런 application은 서비스간의 인증 및 정책 결정을 위해 다양한 시스템과 통신해야합니다.
일반적으로 이런 경우 하드 코딩된 API를 사용하거나 백엔드에서 제공하는 클라이언트를 사용합니다.
이런 통일되지 않는 통신 방식으로 인한 데이터 손실 및 오류는 서비스 전체의 오류로 이어질 수 있습니다.

Istio Mixer는 이런 문제를 해결하기 위해 단일, 통일된 추상화를 사용합니다.
각 요청은 컨텍스트 및 세부 사항을 설명하는 속성 목록으로 표시됩니다.
예를 들어, `source.IP source.user request.path request.size` 등은 요청을 설명하는 속성입니다. 
이 속성 및 속성의 처리는 Mixer 엔진의 핵심 기능입니다. 

Istio는 data plane을 통해 들어오는 각 요청에 대해 이러한 속성을 기록하고 이를 Mixer로 보냅니다.
Mixer가 이 속성을 수신하면, 이 요청에 대한 처리를 진행할지 telemetry 수집할지를 결정합니다.
Mixer는 아래 두 API를 통해 위 동작을 수행합니다.

- Check
- Report

### Check API With ISTIO-POLICY

Check API는 해당 요청이 예상 조건을 만족하는지, 진행 또는 거부해야하는지 여부에 따라 속성을 기반으로 유효성 검증 요청과 함께 인라인으로 호출됩니다.
Mixer는 속성 처리 엔진을 사용하여 속성을 정책 엔진 또는 API 관리 시스템과 같은 특정 백엔드에 대한 요청으로 변환합니다.
Mixer를 사용하면 3rd party에서 속성 처리에 사용하는 어댑터를 생성하고 속성을 백엔드 별 메시지로 작성할 수 있습니다.
이 백엔드 시스템은 긍정 또는 부정으로 응답할 수 있고 API에서 정해진대로 요청을 처리합니다.

![](./images/istio-study-2.2-005.png)
[출처: Istio in Action MEAP Edition]

아래는 Istio Policy 엔진으로 전송되는 속성 샘플입니다.
일반적으로 사용되는 배포 속성은 [Attribute Vocabulary](https://istio.io/docs/reference/config/policy-and-telemetry/attribute-vocabulary/) 에서 확인할 수 있습니다.

![](./images/istio-study-2.2-006.png)

![](./images/istio-study-2.2-007.png)
[출처: Istio in Action MEAP Edition]


### MIXER Report API With ISTIO-TELEMETRY

Check API와 마찬가지로 Report API는 요청에 대한 속성을 전송하는데 사용됩니다.
그러나 Check API와 다르게 비동기적으로 호출되고 요청 경로 밖에 있습니다.

Service proxy가 요청을 처리될때 그 요청의 속성은 proxy에 의해 일괄 처리되고 특정 요청이 들어올 때 또는 일정 시간 후에 Mixer로 전송됩니다.
예를 들어, 100개의 요청이 서비스로 유입되면 service proxy는 그 요청들의 속성을 일괄적으로 처리하고 Mixer로 전송합니다.

![](./images/istio-study-2.2-008.png)
[출처: Istio in Action MEAP Edition]

Control plane에 실제로 Mixer 구성 요소가 배포되어 있진 않습니다. 
Check, Report API는 서로 다른 런타임과 스케일링 조건을 가지기 때문에 분리되어 있습니다.
istio-policy는 Check API를 구현하고 istio-telemetry Report API를 구현합니다.

<p class="tip-title">참고</p>
<p class="tip-content">
Check, Report API는 서로 분리되어 있지만 두 가지 API를 관리하는 Mixer 구성 요소는 하나입니다.
istio-policy, istio-telemetry deployment의 상세 내용을 보면 mixer contrainer를 동일하게 포함하는 것을 확인할 수 있습니다.
</p>



