# Sample 1

## Goal
Host 기반 트래픽 라우팅 설정

<img width="453" alt="스크린샷" src="https://user-images.githubusercontent.com/59307414/236671347-5e658552-bbc7-4c67-a44d-550264bf04fa.png">

`test.wilump.dev:8000`으로 접속 시 nginx 연결

## Setup
`nginx.yaml`: app(service+deployment) + proxy
- `sidecar.istio.io/inject: "true"` 적용

`gateway.yaml`: istio Gateway

`virtual-service.yaml`: istio VirtualService


## Test
로컬호스트 라우팅 확인
```shell
$ cat /etc/hosts                    
##
# Host Database
#
# localhost is used to configure the loopback interface
# when the system is booting.  Do not change this entry.
##
127.0.0.1	localhost
255.255.255.255	broadcasthost
::1             localhost
# Added by Docker Desktop
# To allow the same kube context to work on the host and the container:
127.0.0.1 kubernetes.docker.internal
127.0.0.1 test.wilump.dev
```

Gateway 확인
```shell
$ k get gw 
NAME                     AGE
istio-sample-1-gateway   76s
```

VirtualService 확인
```shell
$ k get vs                       
NAME                             GATEWAYS                     HOSTS                 AGE
istio-sample-1-virtual-service   ["istio-sample-1-gateway"]   ["test.wilump.dev"]   51s
```

## Reference
- https://istio.io/latest/docs/reference/config/networking/virtual-service/