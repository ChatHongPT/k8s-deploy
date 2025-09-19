# 콘서트 티켓팅 웹사이트

국내 콘서트 티켓팅을 위한 웹 기반 플랫폼입니다.

## 기능

- 🎵 콘서트 목록 조회 및 검색
- 🎫 실시간 좌석 선택 및 예매
- 👤 사용자 회원가입 및 로그인
- 💳 간편한 티켓 결제 시스템
- 📱 반응형 웹 디자인
- 🎭 다양한 콘서트 정보 제공

## 요구사항

- Java 17 이상

## 실행 방법

### 1. 프로젝트 빌드
```bash
cd homepage
./gradlew build
```

### 2. 애플리케이션 실행
```bash
./gradlew bootRun
```

### 3. 웹 브라우저에서 접속
```
http://localhost:8080
```

## 주요 화면

- **홈페이지**: 인기 콘서트 및 추천 공연
- **콘서트 목록**: 카테고리별 콘서트 검색
- **콘서트 상세**: 공연 정보 및 좌석 선택
- **예매 시스템**: 실시간 좌석 예약 및 결제
- **마이페이지**: 예매 내역 및 티켓 관리

## 기술 스택

- Spring Boot 3.5.5
- Thymeleaf
- Bootstrap 5
- Lombok
- 메모리 기반 데이터 저장

## 라이선스

MIT License


---


# 🐳 Minikube를 활용한 Spring Boot Kubernetes 배포 가이드

## 개요
이 문서는 Spring Boot 애플리케이션을 로컬에서 빌드한 JAR 파일을 Docker 이미지로 만들고, Kubernetes(Minikube) 클러스터에 배포하는 과정을 설명합니다.

---

## 환경 구성
- **Spring Boot**: 2.x (Gradle 기반)  
- **Ubuntu Server**: 20.04+  
- **Docker**: 20.x 이상  
- **Docker Hub** 계정
- **Minikube** & **kubectl** 설치됨  

---

## 프로젝트 구조

```bash
ubuntu-server:/home/ubuntu/k8s_spring
 ├─ app.jar           # 로컬에서 빌드한 Spring Boot JAR 파일
 ├─ deploy.yaml       # Deployment & Service 정의
 ├─ ingress.yaml      # Ingress 정의
 └─ Dockerfile        # 도커 이미지 빌드용
```

<br> 

## 개념 정리
### Docker
- **Docker**: 컨테이너 기반 애플리케이션 실행 플랫폼
  - 이미지(Image): 컨테이너를 만드는 설계도
  - 컨테이너(Container): 이미지로부터 실행된 인스턴스
- **Docker Hub**: Docker 이미지를 저장·공유하는 클라우드 저장소 (GitHub와 유사 - docker push / docker pull)

### Minikube & Kubernetes
- **Minikube**: 로컬에서 단일 노드 Kubernetes 클러스터를 손쉽게 실행할 수 있는 도구
- **Kubernetes(K8s)**: 컨테이너화된 애플리케이션을 자동 배포·확장·관리하는 오케스트레이션 플랫폼
- **주요 오브젝트**
  - Pod: 가장 작은 실행 단위, 하나 이상의 컨테이너 포함
  - Deployment: Pod를 관리하고 원하는 개수/버전을 유지
  - Service: Pod를 외부 네트워크에 노출, IP 주소가 바뀌더라도 Service의 고정된 IP를 통해 Pod에 접근 가능


### YAML & 선언적 방식
- **YAML**: 사람이 읽기 쉬운 데이터 직렬화 언어
- **선언적 방식** : `replicas: 3`과 같이 “최종 상태”를 정의하면 K8s가 자동으로 해당 상태를 보장

<br>

--- 

## Docker 이미지 생성 및 Push

#### 1. Dockerfile 작성
```
FROM openjdk:17-jdk-slim

WORKDIR /app
COPY ./k8s_demo-0.0.1-SNAPSHOT.jar /app/demo.jar

EXPOSE 8082
CMD ["java", "-jar", "demo.jar"]
```
> 해당 내용으로 jar를 파일을 이미지로 생성

#### 2. 이미지 빌드

```
dockr build -t {이미지명}:1.0 .
``` 

> Dockerfile이 있는 경로로 이동후 이미지 생성 명령어를 수행


#### 3. 태그 추가
```
docker tag {이미지명}:1.0 {dockerhub계정 이름}/{저장위치}:1.0
```
> 이미지를 어디에 올릴건지 지정

<details>
  <summary>Docker 이미지 빌드 & 태그 추가 확인</summary> 
  <img width="744" height="201" alt="image" src="https://github.com/user-attachments/assets/af2e54d2-01b6-4e4c-891d-c1f6eeda97e9" />
</details>



#### 4. Docker Hub에 Push
```
docker push {dockerhub계정 이름}/{저장위치}:1.0
```

<br>

---

## Minikube 설정

#### Minikube 최신 버전 설치 및 다운로드한 바이너리 삭제
```
$ curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

$ sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
```

#### Minikube 버전 확인
```
$  minikube version
```

#### kubectl 설치 하기 (미존재시만)
```
$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

$ sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```


#### kubectl 버전 확인
``` 
$  kubectl version --output=yaml
``` 
  

#### Minikube 시작 
```
$ minikube start
```

---

## 애플리케이션 배포
### Deployment

Spring Boot Pod를 관리하고, 지정한 replica 수만큼 항상 유지

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: springboot
  template:
    metadata:
      labels:
        app: springboot
    spec:
      containers:
      - name: springboot
        image: jangsongha/no_db_spring:1.0
        ports:
        - containerPort: 8082
        env:
        - name: SERVER_PORT
          value: "8082"
```
- replicas: 3 → Pod 3개 실행으로 가용성 확보

- containerPort: 8082 → 컨테이너 내부에서 사용하는 포트

- SERVER_PORT 환경 변수 → Spring Boot 서버 포트 설정 가능

<br>

### Service

Deployment에서 생성한 Pod들을 하나의 접근점으로 묶고, 외부에서 접근할 수 있도록 설정
```
apiVersion: v1
kind: Service
metadata:
  name: springboot-service
spec:
  selector:
    app: springboot
  ports:
    - protocol: TCP
      port: 8082        # 클러스터 내부 포트
      targetPort: 8082  # Pod 컨테이너 포트
      nodePort: 30081   # 외부 접근용 포트
  type: NodePort

```

- type: NodePort → 외부에서 접근 가능

- 외부 접근 URL 예시: http://<Minikube-IP>:30081/

- Service 포트(8082)와 Pod 포트(8082) 일치 필요

<br>

### Ingress

외부 요청을 Service로 라우팅하여, 도메인 기반 접근 가능
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: springboot-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: spring.app
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: springboot-service
            port:
              number: 8082

```

- host: spring.app → 외부에서 접근할 도메인 이름

- annotations: nginx.ingress.kubernetes.io/rewrite-target: / → 요청 경로를 Service로 전달할 때 /로 변환

- backend service: springboot-service의 8082 포트로 라우팅

- pathType: Prefix → / 이하 모든 경로를 Service로 전달

<br>

### 리소스 적용 및 확인
```
kubectl apply -f deploy.yaml
kubectl apply -f ingress.yaml
```
#### 상태 확인:
```
kubectl get pods
kubectl get svc
kubectl get ingress
```
<details>
  <summary>상태 확인 결과</summary> 
  <img width="692" height="144" alt="image" src="https://github.com/user-attachments/assets/6e5225fd-cf27-4864-a22b-078aff2a0f9f" /> 
  <img width="788" height="146" alt="image" src="https://github.com/user-attachments/assets/1833d3b4-1dfa-411b-aec1-c919d1f4ded3" />
</details>



## 전체 흐름 요약
> 1. Spring Boot JAR 빌드
> 2. Dockerfile 작성 → 이미지 빌드
> 3. Docker Hub에 Push
> 4. Minikube 시작
> 5. deploy.yaml, ingress.yaml 적용
> 6. kubectl get으로 리소스 상태 확인

