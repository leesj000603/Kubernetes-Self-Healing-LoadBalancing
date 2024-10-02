# Kubernetes-Self-Healing-LoadBalancing
This project demonstrates the process of setting up a Kubernetes cluster using Minikube and deploying a Spring Boot application based on Docker images to create a service capable of communicating with external resources. It also includes methods for testing load balancing using a load generator and monitoring CPU and memory usage.

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


- ### 클러스터 외부에서 통신이 가능해졌으니 window에서 접속
  Windows → VirtualBox → VM → Minikube → Kubernetes Cluster 의 상태이므로
  <br>
  **1. ubuntu에서 kubernetis에 port forwarding**
```
# Kubernetis의 myapp 서비스의 8080 (spring boot) 을 ubuntu 11111포트에 매핑
kubectl port-forward service/myapp 11111:8080
```


