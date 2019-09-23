# Istio Study #2.1 Kubernetes에 Istio 배포하기

이 문서는 [Istio in Action v5 MEAP](https://www.manning.com/books/istio-in-action) 
내용을 기반으로 작성했습니다. 

이 문서에서는 Istio와 샘플 앱을 배포해보겠습니다.

Istio는 Kubernetes에 의존성이 없고 다양한 플랫폼을 지원합니다.

Kubernetes는 Istio를 배포하기 가장 좋은 플랫폼이므로 이 환경에 배포해보겠습니다.

## 1.Kubernetes 준비하기

Kubernetes Cluster를 minikube, cloud 환경을 활용해 준비합니다.

이 문서에서는
[GKE(Google Kubernetes Engine)](https://cloud.google.com/kubernetes-engine/docs/quickstart?hl=ko)
을 활용했습니다. 구글 계정을 가지고 있다면 Credit을 받아 일정기간 무료로 사용할 수 있습니다.

<p class="tip-title">참고</p>
<p class="tip-content">
이 문서에서는 Kubernetes `v1.13.7-gke.8` 버전을 사용했습니다.
</p>

## 2.Istio 다운로드

이제 Kubernetes 리소스를 활용하여 Istio를 Kubernetes Cluster에 배포해보겠습니다.

[Istio release](https://github.com/istio/istio/releases)에 접속해서 원하는 버전을 선택하고
각자 환경에 맞는 OS의 release를 선택합니다.

<p class="tip-title">참고</p>
<p class="tip-content">
이 문서에서는 1.2.2 버전을 사용했습니다.
</p>

```
wget https://github.com/istio/istio/releases/download/1.2.2/istio-1.2.2-linux.tar.gz
```

다운로드 받은 파일에서 배포 파일을 추출합니다.

```
tar -xzf istio-1.2.2-linux.tar.gz
```

다음으로 배포 파일이 있는 디렉토리를 살펴보겠습니다.

- bin: istioctl 실행 파일
- install: istio 배포를 위한 스크립트 및 리소스 파일
- sample: istio에 배포할 샘플 앱
- tools: istio 테스트 세트

```
$ cd istio-1.2.2
$ ls -l
total 40
drwxr-xr-x  2 ...  4096 Jun 28 02:03 bin
drwxr-xr-x  6 ...  4096 Jun 28 02:03 install
...
drwxr-xr-x 16 ...  4096 Jun 28 02:03 samples
drwxr-xr-x  8 ...  4096 Jun 28 02:03 tools
```

istioctl이 정상 동작하는지 확인합니다.

```
$ ./bin/istioctl version
1.2.2
Error: unable to find any Istio pod in namespace istio-system
```

다음으로 istio를 설치하는데 문제가 없는지 검증합니다.

```
$ ./bin/istioctl verify-install
Checking the cluster to make sure it is ready for Istio installation...
Kubernetes-api
-----------------------
...
Install Pre-Check passed! The cluster is ready for Istio installation.
```

## 3.Istio 구성 요소 설치하기 

Istio를 설치할 때 권장하는 방법은 [Helm](https://helm.sh/)을 사용하는 것입니다.
Helm을 사용하면 Istio 구성 요소의 설정을 상세하게 Customize 할 수 있습니다.

하지만, 이 문서에서는 Customize 없이 모든 구성 요소를 설치하기 위해 Demo Kubernetes 리소스를 사용하겠습니다.
이렇게 설치한 구성 요소는 Helm을 사용해 설치한 것과 동일합니다.

이제 Istio를 설치해보겠습니다. 다운로드한 Istio root 디렉토리로 이동합니다.
Kubectl CLI를 사용해 `istio-demo.yaml` 리소스를 Kubernetes Cluster에 배포합니다.

```
$ kubectl create -f install/kubernetes/istio-demo.yaml
namespace/istio-system created
customresourcedefinition.apiextensions.k8s.io/virtualservices.networking.istio.io created
...
```

배포가 완료되면 istio-system namespace에서 Istio 구성 요소를 조회할 수 있습니다.

```
$ kubectl get po -n istio-system
NAME                                      READY   STATUS      RESTARTS   AGE
grafana-97fb6966d-g6v9w                   1/1     Running     0          95m
istio-citadel-7c7c5f5c99-g5k4s            1/1     Running     0          95m
istio-cleanup-secrets-1.2.2-bcwf6         0/1     Completed   0          95m
istio-egressgateway-f7b8cc667-bj2kd       1/1     Running     0          95m
istio-galley-585fc86678-hb8vn             1/1     Running     0          95m
istio-grafana-post-install-1.2.2-ltg8f    0/1     Completed   0          95m
istio-ingressgateway-cfbf989b7-9zmmg      1/1     Running     0          95m
istio-pilot-68f587df5d-wzwxn              2/2     Running     0          95m
istio-policy-76cbcc4774-8clld             2/2     Running     2          95m
istio-security-post-install-1.2.2-glvhh   0/1     Completed   0          95m
istio-sidecar-injector-97f9878bc-54ssq    1/1     Running     0          95m
istio-telemetry-5f4575974c-gb9gt          2/2     Running     2          95m
istio-tracing-595796cf54-nqh82            1/1     Running     0          95m
kiali-55fcfc86cc-cvw4k                    1/1     Running     0          95m
prometheus-5679cb4dcd-nm4bv               1/1     Running     0          95m
```

Istio는 data plane(service proxy)과 control plane으로 구성됩니다.

Data plane은 application을 배포하지 않았기 때문에 아직 확인할 수 없습니다.
application을 배포하면 service proxy가 pod에 주입됩니다.

Control plane은 아래와 같이 구성됩니다.
Grafana, Prometheus와 같이 익숙한 구성 요소도 있지만 익숙치 않은 구성 요소도 보입니다.
이 구성 요소에 대해서는 뒤에서 좀 더 살펴보도록 하겠습니다.

![](./images/istio-study-2.1-001.png)
[출처: Istio in Action MEAP Edition]

Istio control plane의 구성 요소가 단일 인스턴스로 배포되어 있는 것을 볼 수 있습니다.
이것을 보고 SPOF(Single Point Of Failure)라고 생각할 수 있습니다.
하지만 Istio는 HA(High Availability) 아키텍처를 고려하여 만들어졌습니다.
각 구성 요소는 다중 인스턴스로 배포할 수 있고, 일부 구성 요소에 오류가 있어도 일정 기간 지속될 수 있도록 탄력적으로 만들어져있습니다.

이제 설치가 잘되었는지 정상적인 응답을 반환하는지 확인해보겠습니다.

아래 명령을 실행하여 Istio의 설정을 조회해봅니다.

```
$ kubectl run -i --rm --restart=Never dummy --image=tutum/curl:alpine \
> -n istio-system --command \
> -- curl -v 'http://istio-pilot.istio-system:8080/v1/registration'
```

정상적인 응답이 온다면 아래와 같이 Istio 구성 요소의 endpoint를 확인할 수 있습니다.
이를 통해 우리는 Istio의 control plane에 mesh 설정과 enpoint를 질의할 수 있다는 것을 확인했습니다.

```
...
[
  {
   "service-key": "default-http-backend.kube-system.svc.cluster.local|http",
   "hosts": [
    {
     "ip_address": "10.0.2.8",
     "port": 8080
    }
   ]
  },
  {
   "service-key": "grafana.istio-system.svc.cluster.local|http",
   "hosts": [
    {
     "ip_address": "10.0.0.8",
     "port": 3000
    }
   ]
  },
  ...
```









