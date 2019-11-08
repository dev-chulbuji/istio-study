# Chap06

## Service Resiliency

## Overview

Microservice 환경에서 Application은 다양한 이유로 외부 서비스와의 통신이 불안정하여 서비스의 장애가 발생 할 수 있다.
 - Network 문제
 - 과도한 Traffic 발생으로 인한 연계 Application 장애
 - 연관된 MicroService들의 장애로 인한 계단식 장애


Service Resiliency라는 것은 결국 Application은 장애란 피할 수 없는 요소이며 문제가 발생 했을 때, 빠르게 회복할 수 있게 하자는 Concept이다.

이러한 Concept을 잡고, DevOps(SRE) 측면에서 SLA측정이나 DR 같은 것을 구현 할 수 있도록 하자.

Service Resiliency를 Application에서 Code로 "잘" 구현하는 방법들이 있다. 하지만 "잘" 하기 어렵다.

Istio는 Application에게 Service Resiliency를 위한 구현으로 부터 해방시켜 줄 수 있다. (Decoupling)

### Prerequisite 
앞서 말한 듯이, Application은 완벽할 수 없다. 얼마나 완벽할 수 없는지에 대하여 각 조직별로 아래 지표를 설정하자.

- SLI(Service Level Indicator)
  - 에러율, 가용성, 응답 시간, 처리량, 처리량, 종단간 응답 시간 등
- SLO(Service Level Objectives)
  - ex. Get RPC 호출의 99%(1분 간의 평균)는 100밀리초 이내에 수행되어야 한다.(모든 백엔드 서버에서 측정된 평균 시간이여야 한다.)


### Iteration
1. 시스템의 SLI들을 모니터하고 측정하기
2. SLI를 SLO와 비교해서 별도의 대응이 필요한지 판단하기
3. 대응이 필요한 경우 목표치를 달성하기 위해 어떻게 대응할지 파악하기
4. 대응하기

1~4 반복

다양한 장애 조건을 시뮬레이션하여 장애를 감지하고 복구 할 수있는 방법이 필요. 서비스와 테스트 결과에 어떤 일이 일어나고 있는지 알기 위해서는 모니터링이 아주아주 중요하다. (7장에서 설명: prometheus, kiali, jaeger등을 사용함)


- - -

## Istio(Sidcar Proxy)에서 Service Resiliency

https://istio.io/docs/reference/config/networking/v1alpha3/destination-rule/


 - Client-side load balancing
   - Istio augments Kubernetes out-of-the-box load balancing
 - Timeout
   - Wait only N seconds for a response and then give up
 - Retry
   - If one pod returns an error (e.g., 503), retry for another pod.
 - Circuit breaker
   - Instead of overwhelming the degraded service, open the circuit and reject further requests
 - OutlierDetection(Pool ejection)
   - This provides autoremoval of error-prone pods from the load-balancing pool


## Load Balancing

처리량을 높이고 지연 시간을 줄이는 핵심 기능은로드 밸런싱입니다. 이를 구현하는 간단한 방법은 모든 클라이언트가 통신하고 모든 백엔드 시스템에로드를 분배하는 방법을 알고있는 중앙 집중식로드 밸런서를 보유하는 것입니다. 이것은 훌륭한 접근 방법이지만 병목 현상과 단일 장애 지점이 될 수 있습니다. 클라이언트 측로드 밸런서를 사용하여 클라이언트에로드 밸런싱 기능을 배포 할 수 있습니다. 이러한 클라이언트로드 밸런서는 정교한 클러스터 별로드 밸런싱 알고리즘을 사용하여 가용성을 높이고 대기 시간을 줄이며 전체 처리량을 늘릴 수 있습니다. Istio 프록시에는 다음과 같은 구성 가능한 알고리즘을 통해 클라이언트 측로드 밸런싱을 제공하는 기능이 있습니다.

 - ROUND_ROBIN
   - This algorithm evenly distributes the load, in order, across the endpoints in the load-balancing pool
 - RANDOM
   - This evenly distributes the load across the endpoints in the load-balancing pool but without any order.
 - LEAST_CONN
   - This algorithm picks two random hosts from the load-balancing pool and determines which host has fewer outstanding requests (of the two) and sends to that endpoint. This is an implementation of weighted least request load balancing
- PASSTHROUGH
  - This option will forward the connection to the original IP address requested by the caller without doing any form of load balancing. This option must be used with care. It is meant for advanced use cases. Refer to Original Destination load balancer in Envoy for further details.

라우팅에 대한 이전 장에서는 RouteRules를 사용하여 트래픽이 특정 클러스터로 라우팅되는 방식을 제어하는 것을 보았습니다. 이 장에서는 대상 정책 규칙을 사용하여 특정 클러스터와 통신하는 동작을 제어하는 방법을 보여줍니다. 먼저 Istio **DestinationRule** 규칙으로 로드 밸런싱을 구성하는 방법에 대해 설명합니다.


### DestinationRule Sample

이름 지정을 정의 subset하고 서비스 수준에서 지정된 설정을 재정 의하여 버전 별 정책을 지정할 수 있습니다 . 다음 규칙은 레이블이있는 엔드 포인트 (예 : 포드)로 구성된 testversion이라는 하위 세트로 이동하는 모든 트래픽에 라운드 로빈로드 밸런싱 정책을 사용합니다

    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: bookinfo-ratings
    spec:
      host: ratings.prod.svc.cluster.local
      trafficPolicy:
        loadBalancer:
          simple: LEAST_CONN
      subsets:
      - name: testversion
        labels:
          version: v3
        trafficPolicy:
          loadBalancer:
            simple: ROUND_ROBIN


트래픽 정책은 특정 포트에 맞게 사용자 정의 할 수도 있습니다. 다음 규칙은 포트 80으로의 모든 트래픽에 최소 연결로드 밸런싱 정책을 사용하고 포트 9080으로의 트래픽에 라운드 로빈 로드 밸런싱 설정을 사용

    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: bookinfo-ratings-port
    spec:
      host: ratings.prod.svc.cluster.local
      trafficPolicy: # Apply to all ports
        portLevelSettings:
        - port:
            number: 80
          loadBalancer:
            simple: LEAST_CONN
        - port:
            number: 9080
          loadBalancer:
            simple: ROUND_ROBIN

사용자 쿠키를 해시 키로 사용하여 동일한 등급 서비스에 대한 등급 서비스 해싱 기반로드 밸런서에 대한 고정 세션을 설정

    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: bookinfo-ratings
    spec:
      host: ratings.prod.svc.cluster.local
      trafficPolicy:
        loadBalancer:
          consistentHash:
            httpCookie:
              name: user
              ttl: 0s


*httpHeaderName*

특정 HTTP 헤더를 기반으로 해시.

*httpCookie*

HTTP 쿠키를 기반으로하는 해시

*useSourceIp*

소스 IP 주소를 기반으로 해시

*minimumRingSize*

해시 링에 사용할 최소 가상 노드 수입니다. 기본값은 1024입니다. 더 큰 링 크기는보다 세분화 된 하중 분배를 초래합니다. 로드 밸런싱 풀의 호스트 수가 링 크기보다 크면 각 호스트에 단일 가상 노드가 할당됩니다.


### Circuit breaking & Connection Pool

https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/circuit_breaking

HTTP 요청의 경우 회로 차단으로 인해 x-envoy-overloaded 헤더가 라우터 필터에 의해 설정돔
https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/router_filter#config-http-filters-router-x-envoy-overloaded-set



업스트림 호스트의 연결 풀 설정 설정은 업스트림 서비스의 각 개별 호스트에 적용됩니다. 자세한 내용은 Envoy의 회로 차단기 를 참조 연결 풀 설정은 HTTP 수준뿐만 아니라 TCP 수준에서도 적용 할 수 있음

    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: bookinfo-redis
    spec:
      host: myredissrv.prod.svc.cluster.local
      trafficPolicy:
        connectionPool:
          tcp: # or http
            maxConnections: 100
            connectTimeout: 30ms
            tcpKeepalive:
              time: 7200s
              interval: 75s
              probes: 5
#### TCP
*maxConnections*

대상 호스트에 대한 최대 HTTP1 / TCP 연결 수입니다. 기본값은 1024입니다.

*connectTimeout*

TCP 연결 시간이 초과

*tcpKeepalive*

설정 한 경우 소켓에서 SO_KEEPALIVE를 설정하여 TCP Keepalives를 활성화

#### HTTP
*http1MaxPendingRequests*

대상에 보류중인 최대 HTTP 요청 수입니다. 기본값은 1024입니다.

*http2MaxRequests*

백엔드에 대한 최대 요청 수 기본값은 1024입니다.

*maxRequestsPerConnection*

백엔드 연결 당 최대 요청 수 이 매개 변수를 1로 설정하면 연결 유지가 비활성화됩니다. 기본값은 "무제한"을 의미하며 최대 2 ^ 29입니다.

*maxRetries*

지정된 시간에 클러스터의 모든 호스트에 대해 해결 될 수있는 최대 재시도 횟수입니다. 기본값은 1024입니다.

*idleTimeout*

업스트림 연결 풀 연결에 대한 유휴 시간 종료. 유휴 시간 초과는 활성 요청이없는 기간으로 정의됩니다. 설정하지 않으면 유휴 시간 초과가 없습니다. 유휴 시간 초과에 도달하면 연결이 닫힙니다. 요청 기반 시간 초과는 HTTP / 2 PING이 연결을 유지하지 않음을 의미합니다. HTTP1.1 및 HTTP2 연결에 모두 적용됩니다.

*h2UpgradePolicy*

연관된 대상에 대해 http1.1 연결을 http2로 업그레이드해야하는지 지정하십시오.




### OutlierDetection
업스트림 서비스에서 각 개별 호스트의 상태를 추적하는 회로 차단기 구현. HTTP 및 TCP 서비스 모두에 적용 가능합니다. HTTP 서비스의 경우 API 호출에 대해 5xx 오류를 지속적으로 반환하는 호스트는 사전 정의 된 기간 동안 풀에서 배출됩니다. TCP 서비스의 경우 연속 오류 메트릭을 측정 할 때 지정된 호스트에 대한 연결 시간 초과 또는 연결 실패가 오류로 계산됩니다. 자세한 내용은 Envoy의 이상치 탐지 를 참조하십시오.

다음 규칙은 "검토"서비스에 대한 요구 / 연결이 10 개를 초과하지 않고 100 개의 연결과 1000 개의 동시 HTTP2 요청의 연결 풀 크기를 설정합니다. 또한 5 분마다 업스트림 호스트가 스캔되도록 구성하여 5XX 오류 코드로 7 번 연속 실패하는 호스트는 15 분 동안 배출됩니다.

    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: reviews-cb-policy
    spec:
      host: reviews.prod.svc.cluster.local
      trafficPolicy:
        connectionPool:
          tcp:
            maxConnections: 100
          http:
            http2MaxRequests: 1000
            maxRequestsPerConnection: 10
        outlierDetection:
          consecutiveErrors: 7
          interval: 5m
          baseEjectionTime: 15m


*consecutiveErrors*

연결 풀에서 호스트를 꺼내기 전의 오류 수 HTTP를 통해 업스트림 호스트에 액세스하면 502, 503 또는 504 리턴 코드가 오류로 규정됩니다. 불투명 한 TCP 연결을 통해 업스트림 호스트에 액세스하면 연결 시간 초과 및 연결 오류 / 실패 이벤트가 오류로 간주됩니다.

*interval*

배출 스윕 분석 사이의 시간 간격. 형식 : 1h / 1m / 1s / 1ms. 1ms 이상이어야합니다. 기본값은 10입니다.

*baseEjectionTime*

최소 배출 시간. 호스트는 최소 배출 지속 시간과 호스트 배출 횟수와 같은 기간 동안 배출 상태를 유지합니다. 이 기술을 통해 시스템은 비정상 업스트림 서버의 추출 기간을 자동으로 늘릴 수 있습니다. 형식 : 1h / 1m / 1s / 1ms. 1ms 이상이어야합니다. 기본값은 30입니다.

*maxEjectionPercent*

추출 할 수있는 업스트림 서비스에 대한로드 밸런싱 풀에있는 호스트의 최대 %. 기본값은 10 %입니다.

*minHealthPercent*

연결된로드 밸런싱 풀의 상태가 최소 상태 인 호스트가 정상 모드 인 경우 이상치 탐지가 활성화 됩니다. 로드 밸런싱 풀의 정상 호스트 백분율이이 임계 값 아래로 떨어지면 이상 값 감지가 비활성화되고 프록시는 풀의 모든 호스트 (로드 상태 및 상태)에서로드 밸런스를 수행합니다. 임계 값은 0 %로 설정하여 비활성화 할 수 있습니다. 서비스 당 포드 수가 적은 k8s 환경에서는 일반적으로 적용 할 수 없으므로 기본값은 0 %입니다.


# 이제 Chaos Engineering을 하면 된다.