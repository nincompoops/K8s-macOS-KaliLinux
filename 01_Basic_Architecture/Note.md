# deployment_liveness.yaml

```md
kubectl apply -f 01_Basic_Architecture/deployment_liveness.yaml
```
<img width="520" height="73" alt="Image" src="https://github.com/user-attachments/assets/e51164ce-bd6a-4c87-b89c-f1f493139c9f" />

명령어를 입력하면 이미지처럼 성공적으로 Deployment를 클러스터에 배포했다고 나온다.

<br>

```md
kubectl get pods
```
<img width="523" height="143" alt="Image" src="https://github.com/user-attachments/assets/2c3e08cc-2ac0-4f74-9656-7e67bcde142c" />

이후에 `STATUS`를 보면 이전 Deployment는 Running인 상태로 나온다. 또한 RESTARTS가 1이고 AGE가 4h12m인걸 보아, 4시간 12분동안
1번의 재시작이 있었음을 알 수 있다.
<br>
그러나 바로 직전에 클러스터에 배포한 liveness Deployment는 STATUS가 CrashLoopBackOff라고 나온다. 아래의 명령어를 입력해서 자세한
설명을 보자.

<br>

```md
kubectl describe pod nginx-live
```
```markdawn
┌──(doochul㉿doochul)-[~/K8s-macOS-KaliLinux]
└─$ kubectl describe pod nginx-live
Name:             nginx-liveness-deploy-669ccb498d-crwk7
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.49.2
Start Time:       Tue, 28 Oct 2025 14:57:13 +0900
Labels:           app=nginx-liveness
                  pod-template-hash=669ccb498d
Annotations:      <none>
Status:           Running
IP:               10.244.0.6
IPs:
  IP:           10.244.0.6
Controlled By:  ReplicaSet/nginx-liveness-deploy-669ccb498d
Containers:
  nginx:
    Container ID:   docker://6e6603f8028cbef6c59733f99958bafb9b50aa3ea19a50eedd12366c87ceaf84
    Image:          nginx
    Image ID:       docker-pullable://nginx@sha256:029d4461bd98f124e531380505ceea2072418fdf28752aa73b7b273ba3048903
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Tue, 28 Oct 2025 15:21:48 +0900
      Finished:     Tue, 28 Oct 2025 15:22:06 +0900
    Ready:          False
    Restart Count:  13
    Liveness:       http-get http://:80/health delay=5s timeout=1s period=5s #success=1 #failure=3
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-7bbl6 (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  kube-api-access-7bbl6:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age                   From               Message
  ----     ------     ----                  ----               -------
  Normal   Scheduled  28m                   default-scheduler  Successfully assigned default/nginx-liveness-deploy-669ccb498d-crwk7 to minikube
  Normal   Pulled     28m                   kubelet            Successfully pulled image "nginx" in 1.801s (1.801s including waiting). Image size: 172382727 bytes.
  Normal   Pulled     28m                   kubelet            Successfully pulled image "nginx" in 1.854s (1.854s including waiting). Image size: 172382727 bytes.
  Normal   Pulled     27m                   kubelet            Successfully pulled image "nginx" in 1.85s (1.85s including waiting). Image size: 172382727 bytes.
  Normal   Pulled     27m                   kubelet            Successfully pulled image "nginx" in 1.767s (1.767s including waiting). Image size: 172382727 bytes.
  Normal   Created    27m (x5 over 28m)     kubelet            Created container: nginx
  Normal   Started    27m (x5 over 28m)     kubelet            Started container nginx
  Normal   Pulled     27m                   kubelet            Successfully pulled image "nginx" in 1.783s (1.783s including waiting). Image size: 172382727 bytes.
  Normal   Killing    26m (x5 over 28m)     kubelet            Container nginx failed liveness probe, will be restarted
  Warning  Unhealthy  25m (x18 over 28m)    kubelet            Liveness probe failed: HTTP probe failed with statuscode: 404
  Normal   Pulling    4m27s (x13 over 28m)  kubelet            Pulling image "nginx"
  Warning  BackOff    105s (x104 over 26m)  kubelet            Back-off restarting failed container nginx in pod nginx-liveness-deploy-669ccb498d-crwk7_default(65d842a7-31c1-443a-aa11-82f3ccd3caf8)
```

마지막 `Events` 부분에서 `Reason` 란을 보면,
<br>
`Unhealthy`라는 부분을 볼 수 있다. `Message`에는 `Liveness probe failed: HTTP probe failed with statuscode: 404`라는 문구가 있는데, 이런
문구가 나오는 이유는 아래의 `deployment_liveness.yaml` 파일 내부를 보면 알 수 있다.

```md
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-liveness-deploy
  labels:
    app: nginx-liveness
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-liveness
  template:
    metadata:
      labels:
        app: nginx-liveness
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

`livenessProbe`의 정의를 살펴보자.

```md
livenessProbe:
  httpGet:
    path: /health
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 5
```
YAML 파일의 위의 부분은 **Kubelet**에게 파드 내부 컨테이너의 생존 여부를 주기적으로 어떻게 확인해야 하는지
지시한다.
<br>
해석해보자면, `path` 부분은 **Kubelet**이 컨테이너로 요청할 경로를 나타낸다. `port` 부분은 요청을 보낼 포트를 
지정해주는 것이고 `periodSeconds`는 5초마다 검사를 실시한다는 것을 말한다.
<br>
그러나 Nginx는 `/` 경로나 존재하는 파일에 대해서만 `200 OK` 응답을 보낸다. 하지만, `/health`라는 경로는 
Nginx 서버에 정의되어 있지 않다.
<br>
Liveness Probe는 HTTP 검사 시 응답 코드가 `200` 이상 `400` 미만일 때만 **성공**으로 간주한다.
그러나 위의 결과를 보면 Kubelet이 `path: /health`로 요청을 보낼 때, Nginx는 해당 경로를 찾을 수 없다는 `statuscode: 404`를 반환하는데, 
이것은 Probe는 이 응답을 **컨테이너가 비정상(Unhealthy) 상태**라는 신호로 해석한다.
<br>
최종적으로는 Probe가 설정된 횟수만큼 연속으로 실패하면, **Kubelet**은 Deployment의 선언적 상태(Desired State)를 유지하기 위해
해당 컨테이너를 **강제로 종료(Kill)하고 자동으로 재시작(Restart) 시킨다.**
