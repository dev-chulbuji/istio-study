# Istio Study #2.3 Service Mesh에 첫 번째 application 배포하기

이 문서는 [Istio in Action v5 MEAP](https://www.manning.com/books/istio-in-action) 
내용을 기반으로 작성했습니다. 

## Overview

새로운 스타트업 Conference Outfitters가 웹 사이트와 재고 및 결제를 지원하는 시스템을 개편하고 있습니다.
그들은 Kubernetes를 다음과 같이 사용하기로 결정했습니다.
배포 플랫폼의 및 응용 프로그램을 특정 클라우드 공급 업체가 아닌 Kubernetes API에 구축하려고 합니다.
그들은 서비스간 통신의 몇 가지 문제를 해결하려하고 헤드 아키텍트가 Istio에 대해 알게되었을 때 그것을 활용하기로 결정했습니다.

Conference Outfitter의 application은 일반적인 온라인 웹 스토어입니다. 매장을 구성하는 구성 요소를 살펴 보겠습니다.
먼저 Istio의 기능을 살펴보기 위해 구성 요소의 작은 하위 집합에 중점을 두겠습니다. 
웹 스토어는 AngularJS 및 NodeJS로 만든 web-facing 사용자 인터페이스로 구성됩니다.
이것은 backend의 catalog, inventory 서비스를 관리하는 gateway 서비스와 통신합니다.
첫 번째 예제에서는 gateway와 catalog 서비스만 배포합니다.

![](./images/istio-study-2.3-001.png)
[출처: Istio in Action MEAP Edition]

## 1.예제 소스코드 준비

이 예제의 소스코드를 [book-source-code github](https://github.com/istioinaction/book-source-code)에서 다운로드합니다.
```
git clone https://github.com/istioinaction/book-source-code.git
cd book-source-code
```

book-source-code 디렉토리의 아래 경로에서 catalog 서비스를 조회해봅니다.
```
$ cat services/catalog/kubernetes/catalog.yaml
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: catalog
  name: catalog
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 3000
  selector:
    app: catalog
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: catalog
    version: v1
  name: catalog
spec:
  replicas: 1
  selector:
    matchLabels:
      app: catalog
      version: v1
  template:
    metadata:
      labels:
        app: catalog
        version: v1
    spec:
      containers:
      - env:
        - name: KUBERNETES_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        image: istioinaction/catalog:latest
        imagePullPolicy: IfNotPresent
        name: catalog
        ports:
        - containerPort: 3000
          name: http
          protocol: TCP
        securityContext:
          privileged: false
```

## 2.배포 준비

`istioinaction` namespace를 생성하고 기본 context를 설정합니다.
```
$ kubectl create namespace istioinaction
$ kubectl config set-context $(kubectl config current-context) \
 --namespace=istioinaction
```

Application을 배포하기 전에 Istio service proxy를 injection 해야합니다.
아래와 같이 istioctl를 통해 service proxy를 injection 할 수 있습니다.
결과를 확인해보면 istio proxy container 및 설정이 있는 것을 확인할 수 있습니다.
```
$ istioctl kube-inject -f services/catalog/kubernetes/catalog.yaml
   ...
   image: docker.io/istio/proxyv2:1.2.2
   imagePullPolicy: IfNotPresent
   name: istio-proxy
```

## 3.Catalog 서비스 배포

이제 예제의 catalog 서비스를 배포해보겠습니다.
istioctl를 통해 service proxy를 injection하고 그 결과를 바로 kubectl을 통해 배포합니다.
```
$ kubectl create -f <(istioctl kube-inject \
-f services/catalog/kubernetes/catalog.yaml)

service/catalog created
deployment.apps/catalog created
```

배포 결과를 조회해봅니다. 
최초 PodInitializing 상태에서 일정 시간이 지나면 Running 상태로 변경됩니다.
실제 catlog 서비스의 컨테이너는 1개이지만 service proxy가 injection되었기 때문에 2개의 컨테이너를 확인할 수 있습니다.
```
$ kubectl get pod
NAME                       READY   STATUS            RESTARTS   AGE
catalog-66f64d84df-8l6rp   0/2     PodInitializing   0          50s

...

$ kubectl get po
NAME                       READY   STATUS    RESTARTS   AGE
catalog-66f64d84df-8l6rp   2/2     Running   0          2m47s
```

아래 명령을 실행해서 catalog 서비스가 정상적으로 호출되는지 요청을 해보겠습니다.
정상적으로 동작한다면 아래와 같이 상품 목록을 응답으로 반환해줍니다.
```
$ kubectl run -i --rm --restart=Never dummy \
--image=dockerqa/curl:ubuntu-trusty --command \
-- sh -c 'curl -s catalog/items'
[
 {
 "id": 0,
 "color": "teal",
 "department": "Clothing",
 "name": "Small Metal Shoes",
 "price": "232.00"
 },
 {
 "id": 1,
 "color": "turquoise",
 "department": "Toys",
 "name": "Generic Plastic Sausages",
 "price": "144.00"
 },
 {
 "id": 2,
 "color": "purple",
 "department": "Jewelery",
 "name": "Intelligent Metal Bike",
 "price": "484.00"
 },
...
]
```

## 4.API Gateway 서비스 배포

다음으로 backend 서비스의 facade를 제공하는 API Gateway 서비스를 배포해보겠습니다.

```
$ kubectl create -f <(istioctl kube-inject \
-f services/apigateway/kubernetes/apigateway.yaml)

service/apigateway created
deployment.extensions/apigateway created
```

이전에 배포한 catalog 서비스와 함께 apigateway pod를 확인할 수 있습니다.
```
$ kubectl get pod
NAME                          READY   STATUS    RESTARTS   AGE
apigateway-85d7785b48-h9wfd   2/2     Running   0          3m45s
catalog-66f64d84df-8l6rp      2/2     Running   0          14m
```

apigateway 서비스가 정상적으로 호출되는지 확인합니다.
정상적으로 동작한다면 catalog 서비스를 호출했을 때와 마찬가지로 상품 목록을 응답으로 반환해줍니다.
```
$ kubectl run -i --rm --restart=Never dummy --image=dockerqa/curl:ubuntu-trusty \
--command -- sh -c 'curl -s apigateway/api/catalog'
```

지금까지 catalog 및 apigateway 서비스를 Istio service proxy와 함께 배포했습니다.
각 서비스는 자신의 proxy를 가지고 있고 서비스 간의 모든 traffic은 이 proxy를 통해서 전달됩니다.
![](./images/istio-study-2.3-002.png)
[출처: Istio in Action MEAP Edition]


