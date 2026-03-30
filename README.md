# Docker HealthCheck 기반 컨테이너 의존성 제어 실습

Docker Compose 환경에서 `HEALTHCHECK`를 활용해 컨테이너의 실제 준비 상태를 감지하고, 이를 기반으로 서비스 간 실행 순서를 정밀하게 제어하는 구조를 구축한다.

---

## 핵심 개념: HEALTHCHECK란?

컨테이너가 **"실행 중(Running)"** 이라는 것과 **"실제로 사용 가능한 상태(Healthy)"** 는 다르다.
```
컨테이너 시작
     ↓
프로세스 실행됨  →  Docker 입장: "Running" ✅
     ↓
실제 서비스 준비  →  HEALTHCHECK 통과 시: "Healthy" ✅
```

`HEALTHCHECK` 없이 `depends_on`만 쓰면 컨테이너가 **뜨는 순서**만 제어할 뿐,
내부 소프트웨어가 실제로 **준비됐는지는 보장하지 못한다.**

---

## 📁 프로젝트 구조
```
03.compose/
├── Dockerfile
├── docker-compose.yml
└── *.jar
```

---

## ⚙️ HEALTHCHECK 설정

### DB — `docker-compose.yml`

MySQL은 컨테이너가 뜬 직후 쿼리를 받을 수 없는 초기화 시간이 존재한다.
`mysqladmin ping`으로 실제 쿼리 처리 가능 여부를 10초마다 체크한다.
```yaml
healthcheck:
  test: ["CMD-SHELL", "mysqladmin ping -h localhost -u root -p$$MYSQL_ROOT_PASSWORD || exit 1"]
  interval: 10s      # 10초마다 체크
  timeout: 5s        # 5초 내 응답 없으면 실패
  retries: 10        # 10번 실패 시 unhealthy
  start_period: 30s  # 초기 30초는 실패해도 카운트 안 함
```

### App — `Dockerfile`

Spring Boot가 8080 포트에서 실제로 수신 대기 중인지 TCP 소켓으로 확인한다.
Compose 환경 없이 단일 컨테이너로 실행해도 자체 상태 감시가 동작한다.
```dockerfile
HEALTHCHECK --interval=10s --timeout=30s --retries=3 \
  CMD bash -c "echo > /dev/tcp/localhost/8080" || exit 1
```

> **왜 curl 대신 TCP 소켓?**
> curl은 HTTP 응답 경로가 필요하다. 경로를 특정하기 어려운 경우 포트 오픈 여부만 체크하는 TCP 방식이 더 범용적이다.

---

## ⚙️ 의존성 제어 설정

DB의 HEALTHCHECK 결과를 App 시작 조건으로 연결한다.
```yaml
app:
  depends_on:
    db:
      condition: service_healthy  # DB가 healthy 판정 전까지 App 생성 차단
```
```
DB 컨테이너 시작
     ↓
mysqladmin ping 성공 → healthy 판정
     ↓
App 컨테이너 시작 허용
     ↓
8080 포트 오픈 확인 → healthy 판정
```

---

## 전체 설정 파일

```yaml
services:
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: fisa
      MYSQL_USER: user01
      MYSQL_PASSWORD: user01
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h localhost -u root -p$$MYSQL_ROOT_PASSWORD || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s
    networks:
      - spring-mysql-net

  app:
    image: wooxxo:test-v1
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://db:3306/fisa
      SPRING_DATASOURCE_USERNAME: user01
      SPRING_DATASOURCE_PASSWORD: user01
      SERVER_PORT: 8080
    depends_on:
      db:
        condition: service_healthy
    networks:
      - spring-mysql-net

networks:
  spring-mysql-net:
```

```dockerfile
FROM eclipse-temurin:17-jdk

RUN apt-get update \
 && apt-get install -y curl \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY *.jar app.jar

ENV SERVER_PORT=8080

HEALTHCHECK --interval=10s --timeout=30s --retries=3 \
  CMD bash -c "echo > /dev/tcp/localhost/8080" || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## 🚀 실행 방법
```bash
# 1. 이미지 빌드
docker build -t wooxxo:test-v1 .

# 2. 컨테이너 실행
docker compose up -d

# 3. 상태 확인
docker ps
```

---

## ✅ 검증 결과

### 초기 실행

`docker compose up -d` 직후 — DB가 `healthy`가 된 직후 App이 `starting` 상태로 진입하는 것을 확인.

![초기 실행](/Images/docker-1.png)

> DB(8초 업)가 healthy 판정된 직후 App(2초 업)이 시작됨 — `depends_on: service_healthy` 동작 증거

---

### healthy 안정화

약 1분 후 — DB, App 모두 `healthy` 상태로 전환 완료.

![healthy 안정화](/Images/docker-2.png)

---

| 테스트 시나리오 | 예상 결과 | 실제 확인 사항 |
|---|---|---|
| 초기 실행 | DB `healthy` 후 App 시작 | DB `healthy` 판정 직후 App `Up` 확인 *(위 스크린샷)* |
| 포트 충돌 | `address already in use` 에러 | 3306 포트 점유 프로세스 종료 후 정상 가동 |
| DB 강제 종료 | App 유지, DB 연결 실패 | App 컨테이너 생존, API 호출 시 에러 응답 확인 |
| DB 재시작 | 서비스 자동 복구 | DB 재가동 후 App `healthy` 복구 확인 |

---

## 💡 고찰

**`depends_on` 단독 사용의 한계**
`depends_on`만으로는 컨테이너 생성 순서만 제어된다. `service_healthy` 조건과 결합해야 내부 소프트웨어의 실제 준비 상태까지 보장할 수 있다.

**HEALTHCHECK의 두 가지 역할**
- `docker-compose.yml`의 healthcheck → 다른 서비스의 시작 조건으로 활용
- `Dockerfile`의 HEALTHCHECK → `docker ps`에서 가용 상태를 즉각 파악 가능, Compose 없이도 동작

---

## 🛠️ 트러블슈팅

| 문제 | 원인 | 해결 |
|---|---|---|
| 이미지 빌드 실패 | `openjdk` 공식 이미지 중단 | `eclipse-temurin:17-jdk`로 교체 |
| JAR 복사 실패 | 파일명 불일치 | `COPY *.jar app.jar` 와일드카드 사용 |
| 호스트 포트 충돌 | 3306 포트 선점 | `lsof -i :3306` → `kill -9 {PID}` |
| healthcheck 환경변수 미적용 | `CMD`는 shell을 거치지 않아 변수 치환 불가 | `CMD-SHELL`로 변경 |
