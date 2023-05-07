# istio-in-action
## 환경 구성
### istioctl 설치
Download: https://istio.io/latest/docs/setup/getting-started/#download

```shell
curl -L https://istio.io/downloadIstio | sh -
```
- vi ~/.zshrc에 `export PATH={설치받은 istio의 디렉토리 경로}/bin:$PATH` 추가

#### istioctl 설치 확인
```shell
istioctl 
```

### istio profile 설치
Install: https://istio.io/latest/docs/setup/install/istioctl/

```shell
# profile 설정을 안 하면 default로 설치됨
istioctl install

# e.g. demo 설치
istioctl install --set profile=demo
```

- 보통 default로 설치하고 커스텀하는 방식
- install configuration: https://istio.io/latest/docs/setup/additional-setup/config-profiles/
- manifest 파일 생성: `istioctl manifest generate > generated-manifest.yaml`

### istioOperator를 이용한 환경 구성
Install: https://istio.io/latest/docs/setup/install/operator/

IstioOperator options: https://istio.io/latest/docs/reference/config/istio.operator.v1alpha1/

#### istio operator 적용
```shell
istioctl operator init
```

#### 확인 
`istio-operator`, `istio-system` ns 생성
```shell
$ kubectl get ns
NAME              STATUS   AGE
default           Active   259d
istio-operator    Active   42s 
istio-system      Active   19m
kube-node-lease   Active   259d
kube-public       Active   259d
kube-system       Active   259d
```

`istio-operator`, `istio-system` pod 생성 생성
```shell
$ kubectl get pods -A
istio-operator   istio-operator-5964f5c8d8-sl6mp         1/1     Running   0             2m10s
istio-system     istio-ingressgateway-55df5d8468-xzhwm   1/1     Running   0             21m
istio-system     istiod-7fb4bc46ff-cfmfs                 1/1     Running   0             21m
kube-system      coredns-787d4945fb-vnjtq                1/1     Running   0             33m
kube-system      etcd-minikube                           1/1     Running   0             33m
kube-system      kube-apiserver-minikube                 1/1     Running   0             33m
kube-system      kube-controller-manager-minikube        1/1     Running   0             33m
kube-system      kube-proxy-z7vnh                        1/1     Running   0             33m
kube-system      kube-scheduler-minikube                 1/1     Running   0             33m
kube-system      storage-provisioner                     1/1     Running   3 (32m ago)   259d
```

### istio operator를 설정 파일로 적용
```yaml
# https://istio.io/latest/docs/setup/install/operator/#install-istio-with-the-operator
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: example-istiocontrolplane
spec:
  profile: default
```

### 커스텀하게 리소스 수정하기
예를 들어 ingressgateway의 설정값을 수정하고 싶은 경우

```shell
# pod 검색
$ k get po -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
istio-ingressgateway-55df5d8468-xzhwm   1/1     Running   0          33m
istiod-7fb4bc46ff-cfmfs                 1/1     Running   0          34m

# pod 설정을 yaml로 조회
$ k get po istio-ingressgateway-55df5d8468-xzhwm -n istio-system -o yaml

# 파일로 떨구기
$ k get po istio-ingressgateway-55df5d8468-xzhwm -n istio-system -o yaml > default-istio-ingressgateway.yaml
```

기본적인 설정에 대한 프로필은 istio-x.x.x/manifests/profiles 디렉토리에 있음. 해당 설정 파일을 복붙해서 수정해도 됨

istio operator manifest 파일에 원하는 부분 추가/수정 후 적용

```yaml
# sample-istio-operator.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: example-istiocontrolplane
spec:
  profile: demo
  components:
    ingressGateways:
      - name: istio-ingressgateway
        enabled: true
        k8s:
          resources:
            requests:
              cpu: 100m
              memory: 400Mi
```

## 사이드카 인젝션 적용하기
### Manual하게 deployment에 사이드카 배포(injection)
`istioctl kube-inject` 활용

```shell
$ istioctl kube-inject -f deployment-nginx.yaml | kubectl apply -f -
deployment.apps/nginx-deployment configured
```

#### 확인
```shell
# pod 조회: pod 내 container 1개(nginx)
$ k get po -n default
NAME                              READY   STATUS    RESTARTS   AGE
nginx-deployment-54bbcfcd-9lqn5   1/1     Running   0          53s

# 사이드카 배포
$ istioctl kube-inject -f deployment-nginx.yaml | kubectl apply -f -
deployment.apps/nginx-deployment configured

# pod 조회: pod 내 container 2개(nginx)
$ k get po -n default
NAME                              READY   STATUS    RESTARTS   AGE
nginx-deployment-54bbcfcd-9lqn5   2/2     Running   0          103s

# pod 상세 조회
$ k describe po nginx-deployment-54bbcfcd-9lqn5 -n default
...
Containers:
  nginx:
    Container ID:   docker://5dc7d98593715e770475eda1610283cd5f06207bc94f3ca442734e3a1fa7d095
    Image:          nginx:1.19.6
    Image ID:       docker-pullable://nginx@sha256:8e10956422503824ebb599f37c26a90fe70541942687f70bbdb744530fc9eba4
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sun, 07 May 2023 17:49:31 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-97m2q (ro)
  istio-proxy:
    Container ID:  docker://b64c16436135577d7438cf1b0955a51036cc406f57753ad8b6da1260f4c77a88
    Image:         docker.io/istio/proxyv2:1.16.1
    Image ID:      docker-pullable://istio/proxyv2@sha256:a861ee2ce3693ef85bbf0f96e715dde6f3fbd1546333d348993cc123a00a0290
    Port:          15090/TCP
    Host Port:     0/TCP
    Args:
      proxy
      sidecar
      --domain
      $(POD_NAMESPACE).svc.cluster.local
      --proxyLogLevel=warning
      --proxyComponentLogLevel=misc:error
      --log_output_level=default:info
      --concurrency
      2
    State:          Running
      Started:      Sun, 07 May 2023 17:49:32 +0900
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     2
      memory:  1Gi
    Requests:
      cpu:      10m
      memory:   40Mi
    Readiness:  http-get http://:15021/healthz/ready delay=1s timeout=3s period=2s #success=1 #failure=30
    Environment:
      JWT_POLICY:                    third-party-jwt
      PILOT_CERT_PROVIDER:           istiod
      CA_ADDR:                       istiod.istio-system.svc:15012
      POD_NAME:                      nginx-deployment-54bbcfcd-9lqn5 (v1:metadata.name)
      POD_NAMESPACE:                 default (v1:metadata.namespace)
      INSTANCE_IP:                    (v1:status.podIP)
      SERVICE_ACCOUNT:                (v1:spec.serviceAccountName)
      HOST_IP:                        (v1:status.hostIP)
      PROXY_CONFIG:                  {}
                                     
      ISTIO_META_POD_PORTS:          [
                                         {"containerPort":80}
                                     ]
      ISTIO_META_APP_CONTAINERS:     nginx
      ISTIO_META_CLUSTER_ID:         Kubernetes
      ISTIO_META_INTERCEPTION_MODE:  REDIRECT
      ISTIO_META_MESH_ID:            cluster.local
      TRUST_DOMAIN:                  cluster.local
    Mounts:
      /etc/istio/pod from istio-podinfo (rw)
      /etc/istio/proxy from istio-envoy (rw)
      /var/lib/istio/data from istio-data (rw)
      /var/run/secrets/credential-uds from credential-socket (rw)
      /var/run/secrets/istio from istiod-ca-cert (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-97m2q (ro)
      /var/run/secrets/tokens from istio-token (rw)
      /var/run/secrets/workload-spiffe-credentials from workload-certs (rw)
      /var/run/secrets/workload-spiffe-uds from workload-socket (rw)
...
```

### namespace를 통해 사이드카 배포(injection)
해당 ns에 뜨는 모든 pod들은 istio 사이드카가 자동으로 injection됨
```shell
# 적용
$ kubectl label namespace default istio-injection=enabled

# 제거
$ kubectl label namespace default istio-injection-
```

### label을 통해 사이드카 배포(injection)
ns 내의 모든 pod에 injection하는 것은 비효율적

label을 통해 injection 대상을 설정할 수 있음

```shell
# Enabled
sidecar.istio.io/inject: "true"

# Disabled
sidecar.istio.io/inject: "false"
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        sidecar.istio.io/inject: "true" // pod의 label에 설정
    spec:
      containers:
        - name: nginx
          image: nginx:1.19.6
          ports:
            - containerPort: 80
```