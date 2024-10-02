# Kubernetes-Self-Healing-LoadBalancing
![image](https://github.com/user-attachments/assets/8ea679f8-fe57-40b5-a12f-055a19a43331)

이 프로젝트는 Minikube를 사용하여 Kubernetes 클러스터를 설정하고, Docker 이미지를 기반으로 한 Spring Boot 애플리케이션을 배포하여 클러스터 외부와 통신이 가능한 서비스를 구성한다. 또한, 부하 생성기를 통해 부하 분산을 테스트하고, CPU 및 메모리 사용량을 모니터링하는 방법을 포함하고 있다.


## 1. Kubernetes를 위한 준비

- ### Minikube 설치
```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
```

- ### kubectl 버전 확인
```
kubectl version
```

- ### Cluster 시작
```
minikube start
```
![image](https://github.com/user-attachments/assets/82bdb342-4d3b-4ee0-b6a5-81d3d11c54f4)

- ### MiniKube 상태 확인
```
minikube status
```

- ### (옵션) kubectl 미존재할 경우만 설치
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```


- ### addon들 보기
```
minikube addons list
```

- ### 서버가 켜질때 항상 켜지도록
```
minikube addons enable dashboard
```

- ### dashboard 실행
```
minikube dashboard
```
![image](https://github.com/user-attachments/assets/fb9ecd56-97cc-4106-917a-3ad891e71472)
- 해당 링크로 접속하면 dashboard 조회 가능

---

## 2. 테스트용 SpringBoot application 준비

- ### 해당하는 기사를 선택시 그에 맞는 기사가 나타나도록 간단히 구현하였다.
![image](https://github.com/user-attachments/assets/1bc8c4d7-91ee-4e93-92b4-6843c221214b)
![image](https://github.com/user-attachments/assets/7d247027-b126-4416-b7d0-0977a2336c03)
![image](https://github.com/user-attachments/assets/10af40ea-f374-4e04-a8a5-075bd9703303)


- ### 조회 요청 시 시간과 로그가 남도록 설정해둠.
![image](https://github.com/user-attachments/assets/b123149d-f02c-485d-bf45-09d5f1ea4b4a)

- ### dockerfile
```
# dockerfile

FROM openjdk:17-slim
# 작업 디렉토리 설정
WORKDIR /app

# 애플리케이션 JAR 파일을 컨테이너의 /app 디렉토리로 복사
COPY myApp-0.0.1-SNAPSHOT.jar /app/app.jar


# 애플리케이션 실행
ENTRYPOINT ["java", "-jar", "app.jar"]
```

- ### SpringBoot Project docker이미지 생성 및 dockerhub저장
```
docker build -t leesj000603/myapp:latest .
docker push leesj000603/myapp:latest
docker pull leesj000603/myapp:latest
```
![image](https://github.com/user-attachments/assets/4ee1cf79-3157-4014-82f8-ad87e4e4a58b)

---

## 3. Springboot pod 생성 및 로드밸런서 서비스 생성


- ###  myapp 애플리케이션의 3개의 파드(Pod)를 생성하여 클러스터에 배포
```
kubectl create deployment myapp --image=leesj000603/myapp --replicas=3
```

- ### dashboard 조회
![image](https://github.com/user-attachments/assets/4fe94841-af86-46eb-987a-c8b9b7432202)
![image](https://github.com/user-attachments/assets/31978f8c-337a-4a5e-8f2c-6dce2d07c603)

- ### cpu 사용량과 메모리 사용량이 표시 안되는 문제
```
minikube addons enable metrics-server
```
**metrics-server addon을 enable하여 pod를 모니터링 할 수 있도록 하였다.**

![image](https://github.com/user-attachments/assets/321a9129-b050-4b14-a4d0-6542ead32d5b)


- ### myapp이라는 이름의 Deployment를 Service로 외부에 노출 하도록 한다. 또한 생성되는 서비스의 유형을 LoadBalancer로 설정
```
kubectl expose deployment myapp --type=LoadBalancer --port=8080
```

- ### 클러스터 외부에서 통신할 수 있는 ip 할당
```
minikube tunnel
```
![image](https://github.com/user-attachments/assets/e9793c53-8341-4409-a798-411ffb2f772b)

![image](https://github.com/user-attachments/assets/1d5f70ef-000a-4273-9b76-c88db715fa26)


- ### 클러스터 외부에서 통신이 가능해졌으니 window에서 접속 해보자
  Windows → VirtualBox → VM → Minikube → Kubernetes Cluster 의 상태이므로
  <br>
  **1. ubuntu vm, kubernetis port forwarding**
  ```
  # Kubernetis의 myapp 서비스의 8080 (spring boot) 을 ubuntu 11111포트에 매핑
  kubectl port-forward service/myapp 11111:8080
  ```

  **2. vm들이 사용하는 NAT 네트워크 포트포워딩 설정 1111 : 1111 으로 매핑**
![image](https://github.com/user-attachments/assets/f36f8af7-2de2-47f0-9402-2406d43694fc)
  **3. 윈도우에서 접속!**
![image](https://github.com/user-attachments/assets/6a97bc9f-44a7-4c0e-b131-c9c7a62d5060)
![image](https://github.com/user-attachments/assets/0bfb4d70-d440-419f-aca3-c8432cdab772)

---

## 4. 부하 분산 여부 확인

```
# 각 pod들의 로그를 통해 pod들에게 요청이 분산 되었는지 확인
kubectl logs [podname]
```

- ### 그러나 분산되지 않음. local에서 요청을 날리는 것 보다 제대로 된 테스트가 필요하다.
  
---

## 5. 부하생성기 pod 생성

- ### load-generator.yml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: load-generator
spec:
  replicas: 3  # pod 3개 생성
  selector:
    matchLabels:
      app: load-generator
  template:
    metadata:
      labels:
        app: load-generator
    spec:
      containers:
      - name: fortio
        image: fortio/fortio
        command: ["fortio", "load", "-c", "100", "-n", "1000", "http://10.110.107.181:8080/article1"]  # 원하는 URL로 부하 생성

```

- ### 부하생성기 실행
```
# 부하 생성기를 정의한 yml파일 실행함으로써 pod 할당
kubectl apply -f load-generator.yaml
```
![image](https://github.com/user-attachments/assets/253e5ad6-6889-4b54-8ac6-797aaff3ab84)

- 부하생성기가 활동하고,
- 그에 따라 myapp pod들의 cpu 사용량 증가


- ### 부하 분산 여부 확인
```
kubectl logs pod/myapp-55f9657889-l5xbf

kubectl logs pod/myapp-55f9657889-9x5m4

kubectl logs pod/myapp-55f9657889-nrdxl


각 springboot application pods의 로그 확인
```
![image](https://github.com/user-attachments/assets/1b39688a-2f59-48aa-bbdd-d6e05b32fa98)

모든 pod에서 부하생성기에서 정의했던 
<br>
command: ["fortio", "load", "-c", "100", "-n", "1000", "http://10.110.107.181:8080/article1"] 에 따라
<br>
article1에 대한 get요청이 날아가고 있음을 확인했다. 다만 엄청난 양의 요청으로 인해 사진으로 구분이 불가능할 뿐 
<br>
모든 pod에 요청이 나눠서 가고 있다.

```
# pod들의 부하 확인
kubectl top pods
```
![image](https://github.com/user-attachments/assets/2d156e20-b192-41cc-88ff-415ef43c706b)


![image](https://github.com/user-attachments/assets/925b66a1-0e92-4272-a404-d59b067e2bb5)


![image](https://github.com/user-attachments/assets/361156ed-11c3-43a9-935f-85a997c81bef)
![image](https://github.com/user-attachments/assets/059ed35d-154a-4151-bb0f-a01571e5bc2f)

---

## 6. Self-Healing

- ### 1. Pod 자동 재시작 (Pod Restart)

Pod는 Kubernetes에서 컨테이너를 실행하는 기본 단위이다. 만약 Pod 내의 컨테이너가 실패하거나 비정상적인 상태로 변경되면, Kubernetes는 자동으로 해당 Pod를 재시작하여 시스템을 복구한다. kubectl get pods 명령어를 통해 확인할 수 있는 RESTARTS 수치는 해당 Pod가 재시작된 횟수를 나타낸다.

예를 들어, restartPolicy가 Always로 설정된 경우, Pod 내의 컨테이너가 비정상적으로 종료되면 Kubernetes가 자동으로 재시작한다.

- ### 2. ReplicaSet & Deployment

ReplicaSet과 Deployment는 여러 개의 동일한 Pod를 관리하는 컨트롤러이다. 특정 Pod가 실패하면, ReplicaSet은 정의된 수의 복제본을 유지하기 위해 자동으로 새로운 Pod를 생성한다. 예를 들어, Deployment에서 3개의 복제본을 설정한 경우, 하나의 Pod가 실패하면 Kubernetes는 새로운 Pod를 생성하여 3개의 복제본이 항상 유지되도록 한다.

- ### loadgenerator가 과부하로 down됐었는데 해당 방식에 의해 바로 다시 살아난 것을 볼 수 있다.
![image](https://github.com/user-attachments/assets/aa279591-35e4-4e79-b960-1ae991103fa9)
![image](https://github.com/user-attachments/assets/0ac2e3cb-43cf-4305-b073-118d49d57a2c)
![image](https://github.com/user-attachments/assets/98b49b6b-9886-49f0-954f-18654e3b94b5)



