# Istio Study #2.4 Isiio observability, resiliency, traffic routing

이 문서는 [Istio in Action v5 MEAP](https://www.manning.com/books/istio-in-action) 
내용을 기반으로 작성했습니다. 

## Overview

지금까지 두 개의 서비스를 배포했습니다. 하지만 외부에서 서비스로 접속할 수 있는 방법이 없습니다. 
Kubernetes 환경에서는 일반적으로 Ingress resource 또는 API Gateway(e.g Gloo)를 사용합니다.
Istio를 사용한다면 Istio Gateway를 사용할 수 있습니다.

<p class="tip-title">참고</p>
<p class="tip-content">
Ingress resource는 Enterprise workload에 적합하지 않습니다. 
그 이유와 Istio Gateway가 이 문제를 어떻게 해결할 수 있는지에 대해서는 추후 살펴보겠습니다. 
</p>

## 1.Istio Gateway 배포하기

apigateway 서비스를 외부에서 접속가능하도록 노출시키기 위해 Istio Gateway를 배포하겠습니다.
```
$ kubectl create -f chapters/chapter2/ingress-gateway.yaml
gateway.networking.istio.io/coolstore-gateway created
virtualservice.networking.istio.io/apigateway-virtualservice created
```

Public cloud 환경에서 Kubernetes cluster를 운영하고 있다면 public cloud의 외부 접속 경로를 아래와 같이 확인할 수 있습니다.
```
$ URL=$(kubectl -n istio-system get svc istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
$ echo $URL
xxx.xxx.xxx.xxx
```

이제 외부에서 apigateway 서비스를 호출합니다.
정상적으로 동작한다면 이전과 마찬가지로 상품 목록이 응답으로 반환됩니다.
```
$ curl $URL/api/catalog
```

## 2.Istio observability

Istio proxy는 요청을 발신/수신하는 양쪽 모두에 위치해 있기 때문에 많은 양의 telemetry를 수집할 수 있습니다.

### Top Level Metrics

먼저 Grafana에 접속해보겠습니다.
Grafana pod로 port forwarding을 설정합니다. 
```
$ GRAFANA=$(kubectl -n istio-system get pod | grep -i running | \
grep grafana | cut -d ' ' -f 1)
$ kubectl port-forward -n istio-system $GRAFANA 8080:3000
```

웹 브라우저에서 `localhost:8080`로 접속합니다.
가장 먼저 Istio dashboard의 리스트를 확인할 수 있습니다.
![](./images/istio-study-2.4-001.png)
[출처: Istio in Action MEAP Edition]

리스트에서 Istio Service Dashboard를 살펴보겠습니다.
Service combo box에서 apigateway.istioinaction.svc.cluster.local를 선택합니다.
하단의 그래프에서 Client Request Volume과 Client Success Rate metric을 확인할 수 있습니다.
이 값은 보통 비어있거나 "N/A" 상태입니다. 
![](./images/istio-study-2.4-002.png)
[출처: Istio in Action MEAP Edition]

터미널에서 서비스에 traffic을 몇 차례 요청합니다. 
```
$ while true; do curl $URL/api/catalog; sleep .5; done
```

Grafana dashboard로 돌아와보면 아래와 같이 metric에 변화가 있는 것을 확인할 수 있습니다. 
서비스 호출이 100% 성공했다는 것을 알 수 있고 P50, P90, 및 P99 tail latencies를 볼 수 있습니다.
![](./images/istio-study-2.4-003.png)
[출처: Istio in Action MEAP Edition]

여기서 중요한 점은 metric을 확인하기 위해 application 코드에 아무것도 추가하지 않았다는 것입니다.

### Distributed Tracing With Opentracing

이번에는 분산 tracing을 다뤄보겠습니다. Istio는 opentracing을 구현한 Jaeger를 기본으로 설치합니다.

Jaeger에 접속하기 위해 istio-tracing pod로 port forwarding을 합니다.
```
$ TRACING=$(kubectl -n istio-system get pod | grep istio-tracing | cut -d ' ' -f 1)
$ kubectl port-forward -n istio-system $TRACING 8181:16686
```

웹 브라우저에서 `http://localhost:8181/`에 접속합니다.
Service combo box에서 istio-ingressgateway를 선택합니다.
그리고 Find Traces 버튼을 선택합니다.
![](./images/istio-study-2.4-004.png)
[출처: Istio in Action MEAP Edition]

만약 아무 것도 나타나지 않는다면 traffic을 발생시키기 위해 서비스를 몇 회 호출합니다.
```
$ while true; do curl $URL/api/catalog; sleep .5; done
```

다시 Jaeger dashboard로 돌아와서 Find Traces 버튼을 선택합니다.
분산 tracing span이 생성된 것을 확인할 수 있습니다. 
![](./images/istio-study-2.4-005.png)
[출처: Istio in Action MEAP Edition]

Span을 선택하면 특정 요청에 대해 상세한 정보를 확인할 수 있습니다.
![](./images/istio-study-2.4-006.png)
[출처: Istio in Action MEAP Edition]

## 3.Istio resiliency

예제 소스코드의 아래 스크립트를 실행하면 모든 요청이 HTTP 500 에러가 발생하도록 설정됩니다. 
```
$ ./bin/chaos.sh 500 100
```

실제로 apigateway 서비스를 호출하면 500에러가 발생하는것을 확인할 수 있습니다.
```
$ curl -v $URL/api/catalog
* Trying 192.168.64.67...
* TCP_NODELAY set
* Connected to 192.168.64.67 (192.168.64.67) port 31380 (#0)
> GET /api/catalog HTTP/1.1
> Host: 192.168.64.67:31380
> User-Agent: curl/7.54.0
> Accept: */*
< HTTP/1.1 500 Internal Server Error
< content-type: text/plain; charset=utf-8
< x-content-type-options: nosniff
< date: Wed, 17 Apr 2019 00:13:16 GMT
< content-length: 30
< x-envoy-upstream-service-time: 4
< server: istio-envoy
<
error calling Catalog service
* Connection #0 to host 192.168.64.67 left intact
```

이번에는 apigateway 서비스를 호출할 떄 50%의 에러가 발생하도록 설정하겠습니다.
```
$ ./bin/chaos.sh 500 50
```

apigateway 서비스를 반복적으로 호출해보면 성공 또는 실패 메세지가 간헐적으로 발생하는 것을 확인할 수 있습니다.
호출 실패의 경우 apigateway 서비스가 catalog 서비스를 호출할 때 발생합니다.
```
$ while true; do curl $URL/api/catalog; \
sleep .5; done
```

이제 Istio가 어떻게 apigateway와 catalog 서비스간의 통신을 더 복원력있게 만들 수 있는지 확인해보겠습니다.

먼저 Istio의 VirtualService 리소스를 살펴보겠습니다.
VirtualService는 서비스간의 상호작용을 정의합니다.
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
 name: catalog
spec:
 hosts:
 - catalog
 http:
 - route:
 - destination:
 host: catalog
 retries:
 attempts: 3
 perTryTimeout: 2s
```

이제 위에서 살펴본 VirtualService 리소스를 배포합니다.
```
$ kubectl apply -f chapters/chapter2/catalog-virtualservice.yaml
virtualservice.networking.istio.io/catalog created
```

apigateway 서비스를 반복적으로 호출해보면 실패 메세지가 더 이상 발생하지 않는 것을 확인할 수 있습니다.
```
$ while true; do curl $URL/api/catalog; \
sleep .5; done
```

여기서 중요한 점은 application 코드를 수정하지 않고 서비스간 통신을 더 복원력있게 만들었다는 점입니다.


## 4.Istio traffic routing