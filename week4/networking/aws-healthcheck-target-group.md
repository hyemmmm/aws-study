# AWS Health Check & Target Group 개념 정리

## 1. Target Group이란?

**Target Group**은 ELB(Elastic Load Balancer)가 트래픽을 전달할
**대상(Target)** 들의 논리적 묶음입니다.

### Target Group의 역할

-   트래픽 전달 대상 정의 (EC2, ECS, Lambda, IP 등)
-   Health Check 수행 주체
-   로드밸런싱 정책 적용 단위

### 구조적 관계

    Client → Load Balancer → Target Group → Target

하나의 Load Balancer는 여러 Target Group을 가질 수 있으며, 리스너 규칙에
따라 서로 다른 Target Group으로 트래픽을 분기할 수 있습니다.

------------------------------------------------------------------------

## 2. AWS Health Check 개념

Health Check는 **"이 Target이 지금 트래픽을 받아도 되는 상태인가?"** 를
판단하는 메커니즘입니다.

### Health Check 흐름

1.  Load Balancer가 주기적으로 Target에 요청 전송
2.  응답 코드 및 시간 확인
3.  조건 충족 시 `Healthy`, 실패 시 `Unhealthy`
4.  Unhealthy Target은 트래픽에서 자동 제외

### Health Check 주요 설정 값

-   **Protocol**: HTTP / HTTPS / TCP
-   **Path**: `/health` 와 같은 엔드포인트
-   **Interval**: 체크 주기 (예: 30초)
-   **Timeout**: 응답 대기 시간
-   **Healthy / Unhealthy Threshold**

------------------------------------------------------------------------

## 3. `/health` 엔드포인트 설계 가이드

### 좋은 `/health` 설계 원칙

#### 1) 빠를 것

-   DB 풀스캔 ❌
-   외부 API 호출 ❌
-   단순 상태 체크 ⭕

#### 2) 항상 동일한 응답

``` http
200 OK
{
  "status": "UP"
}
```

#### 3) 인증 불필요

-   인증/인가 로직 포함 ❌
-   내부 전용 엔드포인트 ⭕

#### 4) 비즈니스 로직과 분리

-   주문 생성, 캐시 워밍 등 ❌
-   서버 생존 여부만 체크 ⭕

#### 5) 장애 전파 최소화

-   하위 의존성(DB, MQ) 상태에 따라 단계적 판단

------------------------------------------------------------------------

# 실무에서 Unhealthy 상태가 빈번하게 발생하는 주요 원인 TOP 5

## 1️⃣ /health 엔드포인트에서 데이터베이스를 직접 점검하는 경우

### 상황
- 일시적인 DB 커넥션 풀 고갈
- `/health` 요청 시 DB 연결 시도
- DB 연결 실패로 인해 Health Check 응답 실패
- ELB가 해당 타겟을 Unhealthy로 판단

### 결과
- 실제 애플리케이션 프로세스는 동작 중
- 트래픽이 즉시 차단됨
- 오히려 서비스 복구 시간이 지연됨

### 권장 대응 방안
- 데이터베이스는 **Soft Dependency**로 취급
- 완전 장애 상태에서만 Health Check 실패를 반환
- 단기적인 커넥션 문제로는 Unhealthy 처리하지 않도록 설계

---

## 2️⃣ Health Check 타임아웃 설정이 과도하게 짧은 경우

### 상황
- Health Check Timeout이 2초로 설정
- GC, 순간적인 부하로 응답 지연 발생 (예: 2.1초)
- 연속 실패로 인해 Unhealthy 판정

### 결과
- 일시적인 성능 저하에도 불필요한 트래픽 차단 발생

### 권장 대응 방안
- Timeout은 **평균 응답 시간의 3~5배 이상**으로 설정
- Interval, Healthy/Unhealthy Threshold와 함께 종합적으로 조정
- 단기 스파이크에 대한 내성 확보

---

## 3️⃣ 애플리케이션 기동 완료 이전에 Health Check가 유입되는 경우

### 상황
- EC2 인스턴스는 기동 완료
- 애플리케이션은 아직 초기화 중
- Health Check 요청이 먼저 도착
- 정상 응답 불가로 Unhealthy 판정

### 결과
- 실제로는 정상 기동 중인 인스턴스가 제외됨

### 권장 대응 방안
- 애플리케이션이 완전히 준비되기 전까지 `/health`는 실패 반환
- 초기화 완료 이후에만 200 응답
- **Ready / Live 개념 분리**를 고려한 설계 권장

---

## 4️⃣ 보안 설정 문제 (Security Group / NACL)

### 상황
- ALB Security Group에서 EC2 Security Group으로 인바운드 허용 누락
- Health Check 요청 자체가 타겟에 도달하지 못함

### 결과
- 실제 서비스 상태와 무관하게 지속적인 Unhealthy 판정

### 권장 대응 방안
- ALB Security Group → Target Security Group 인바운드 허용 필수
- CIDR 기반 허용보다는 **Security Group 참조 방식** 권장
- NACL 설정도 함께 점검

---

## 5️⃣ 리다이렉트 또는 인증 로직이 Health Check에 개입하는 경우

### 상황
- `/health` 요청이 로그인 페이지로 리다이렉트
- 또는 인증 실패로 401/403 응답 반환

### 결과
- Health Check 실패로 Unhealthy 판정

### 권장 대응 방안
- `/health` 엔드포인트는 인증 및 인가에서 완전 예외 처리
- 필터, 인터셉터, 미들웨어 적용 대상에서 제외
- 단순한 200 / 500 응답만 반환하도록 구성

---

## 6️⃣ Health Check와 Auto Scaling의 관계

ELB Health Check는 단순한 모니터링 수단이 아닙니다.

Auto Scaling Group과 연동될 경우 다음과 같은 흐름으로 동작합니다.

1. Target이 Unhealthy 상태로 판단됨
2. ASG에서 인스턴스를 비정상 상태로 인식
3. 인스턴스 종료
4. 신규 인스턴스 생성
5. Health Check 통과 후 트래픽 재투입

👉 **Health Check는 Self-healing 메커니즘의 핵심 트리거**입니다.


------------------------------------------------------------------------

## 6. 한 줄 요약

> **Target Group은 트래픽 전달 단위, Health Check는 생존 판단 기준이다.\
> `/health` 는 가볍고, 빠르고, 예측 가능해야 한다.**
