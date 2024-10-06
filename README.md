# Kubernetes-Self-Healing-LoadBalancing
![image](https://github.com/user-attachments/assets/8ea679f8-fe57-40b5-a12f-055a19a43331)

<img src="https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white"/> <img src="https://img.shields.io/badge/Springboot-6DB33F?style=for-the-badge&logo=Springboot&logoColor=white"/> <img src="https://img.shields.io/badge/Thymeleaf-005F0F?style=for-the-badge&logo=thymeleaf&logoColor=white"/> <img src="https://img.shields.io/badge/VirtualBox-183A61?logo=virtualbox&logoColor=white&style=for-the-badge"/>  <img src="https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white"/>

이 프로젝트는 Minikube를 사용하여 로컬 환경에서 단일 노드 Kubernetes 클러스터를 설정하고, Docker 이미지를 기반으로 한 Spring Boot 애플리케이션을 3개의 pod로 배포하여 클러스터 외부와 통신할 수 있는 부하 분산 서비스를 구성하였다. 또한, 부하 생성기를 사용해 부하 분산 성능을 테스트하고, CPU 및 메모리 사용량을 모니터링하였다.

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


- ### myapp이라는 이름의 Deployment를 Service로 외부에 노출 하도록 한다. 또한 생성되는 서비스의 유형을 LoadBalancer로 설정 - 이를 통해 해당 Service를 통해 들어오는 트래픽을 3개의 pod에 분산하게 된다.
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

  **2. vm들이 사용하는 NAT 네트워크 포트포워딩 설정 11111 : 11111 으로 매핑**
![image](https://github.com/user-attachments/assets/f36f8af7-2de2-47f0-9402-2406d43694fc)
  **3. 윈도우에서 접속!**
![image](https://github.com/user-attachments/assets/6a97bc9f-44a7-4c0e-b131-c9c7a62d5060)
![image](https://github.com/user-attachments/assets/0bfb4d70-d440-419f-aca3-c8432cdab772)
  
---

## 4. 부하생성기 deployment 생성

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
### 주요 요소 설명
- replicas:
  3개의 Pod를 생성하여 부하를 동시에 발생시킨다는 의미이다. 이 설정으로 인해 각 Pod가 지정된 부하 테스트를 병렬로 실행할 수 있다.
- selector:
  matchLabels를 통해 이 Deployment가 생성한 Pod를 찾는 기준이 되는 레이블을 정의한다. 이 레이블은 app: load-generator이다.
- template:
  template 아래에 Pod의 구성을 정의한다. 이 섹션은 Pod의 메타데이터와 스펙을 포함한다.
  - containers:
  Fortio 이미지를 사용하여 부하 생성기를 설정한다. fortio 컨테이너는 Fortio를 실행하며, 지정된 명령으로 요청을 보낸다.
  - command: ["fortio", "load", "-c", "100", "-n", "1000", "http://10.110.107.181:8080/article1"]는 Fortio에 다음과 같은 파라미터를 전달한다:
    -c 100: 동시 연결 수를 100으로 설정한다.
    -n 1000: 총 1000개의 요청을 보낸다.
    - http://10.110.107.181:8080/article1: 부하를 생성할 URL이다.
      - 10.110.107.181은 로드밸런서 서비스의 외부 IP
      - 8080은 Spring application의 port에 해당한다.
      - /article1은 1번 기사를 조회하는 명령이다.
   


### 부하 분산의 원리
- 이 구성에서 부하 생성기 Pod가 3개 생성되므로, 각 Pod가 독립적으로 요청을 보내게 돤다.
  다음과 같은 요청 패턴이 형성된다:

  - Pod 1: 첫 100개의 요청
  - Pod 2: 다음 100개의 요청
  - Pod 3: 마지막 100개의 요청
  <br>
  
  **이렇게 각 Pod가 다른 요청을 동시에 발생시키므로, 전체적으로 로드 밸런서에 전달되는 요청이 분산돤다.**

    


- ### 부하생성기 실행
```
# 부하 생성기를 정의한 yml파일 실행함으로써 3개의 pod를 관리하는  deployment 생성
kubectl apply -f load-generator.yml
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

## 5. Self-Healing

- ### 1. Pod 자동 재시작 (Pod Restart)

Pod는 Kubernetes에서 컨테이너를 실행하는 기본 단위이다. 만약 Pod 내의 컨테이너가 실패하거나 비정상적인 상태로 변경되면, Kubernetes는 자동으로 해당 Pod를 재시작하여 시스템을 복구한다. kubectl get pods 명령어를 통해 확인할 수 있는 RESTARTS 수치는 해당 Pod가 재시작된 횟수를 나타낸다.

예를 들어, restartPolicy가 Always로 설정된 경우, Pod 내의 컨테이너가 비정상적으로 종료되면 Kubernetes가 자동으로 재시작한다.

- ### 2. ReplicaSet & Deployment

ReplicaSet과 Deployment는 여러 개의 동일한 Pod를 관리하는 컨트롤러이다. 특정 Pod가 실패하면, ReplicaSet은 정의된 수의 복제본을 유지하기 위해 자동으로 새로운 Pod를 생성한다. 예를 들어, Deployment에서 3개의 복제본을 설정한 경우, 하나의 Pod가 실패하면 Kubernetes는 새로운 Pod를 생성하여 3개의 복제본이 항상 유지되도록 한다.

- ### loadgenerator가 과부하로 down됐었는데 해당 방식에 의해 바로 다시 살아난 것을 볼 수 있다.
![image](https://github.com/user-attachments/assets/aa279591-35e4-4e79-b960-1ae991103fa9)
![image](https://github.com/user-attachments/assets/0ac2e3cb-43cf-4305-b073-118d49d57a2c)
![image](https://github.com/user-attachments/assets/98b49b6b-9886-49f0-954f-18654e3b94b5)



## 주의사항!

### 1. 세션 관리
- 해결책:
  - Sticky Session: 특정 사용자 요청을 항상 동일한 파드로 라우팅하여 세션 정보를 유지하는 방법이다. 이 방식은 쿠키 기반으로 구현할 수 있으며, 로드밸런서의 유연성을 감소시킬 수 있다.
  - 세션 스토리지 중앙화: Redis와 같은 외부 저장소를 사용하여 세션 정보를 중앙에서 관리할 수 있다. 모든 파드가 이 저장소에 접근하여 세션 정보를 읽고 쓸 수 있으므로 세션 일관성을 유지할 수 있다.
### 2. 데이터 일관성
- 문제: 여러 파드가 동시에 데이터에 접근하고 수정하는 경우, 데이터 불일치 문제가 발생할 수 있다. 예를 들어, 하나의 파드에서 데이터를 업데이트하고 다른 파드에서 그 데이터를 조회하는 경우, 최신 정보가 반영되지 않을 수 있다.
- 해결책:
  - 트랜잭션 관리: 데이터베이스 트랜잭션을 잘 설계하여 데이터 일관성을 유지하는 것이 중요하다. 이를 위해 ACID(Atomicity, Consistency, Isolation, Durability) 특성을 지켜야 한다.
  - 분산 락: ZooKeeper나 Redis를 사용하여 데이터에 대한 분산 락을 관리하면, 특정 시점에서 하나의 파드만 데이터에 접근하도록 할 수 있다.
### 3. 리소스 관리
- 문제:
  Kubernetes는 수요에 따라 파드를 스케일 업/다운할 수 있다. 그러나 이는 리소스 사용이 불규칙해질 수 있으며, 특정 순간에 과도한 리소스를 요구할 수 있다. 부하가 예상치 못하게 증가하면, 적절한 리소스가 할당되지 않아 서비스 장애가 발생할 수 있다.
- 해결책:
  - 모니터링: Prometheus, Grafana와 같은 도구를 사용하여 리소스 사용량을 모니터링하여 스케일링 정책을 조정할 수 있다.
  - 쿼터 및 리미트 설정: 각 파드에 대해 리소스 제한을 설정하여 과도한 리소스 사용을 방지할 수 있다.
