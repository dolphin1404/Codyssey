# Codyssey 개발 워크스테이션 설계 문서

## 1. 개요

본 문서는 AI/SW 개발 워크스테이션의 설계 구조와 동작 원리를 기술합니다.
터미널, Docker, Git을 활용한 재현 가능한 개발 환경의 설계 기준과 의사결정 근거를 정리합니다.

---

## 2. 시스템 아키텍처

```
┌─────────────────────────────────────────────────────┐
│                    Host (Windows 11)                  │
│                                                       │
│  ┌─────────────┐  ┌──────────────────────────────┐   │
│  │  Git Bash    │  │      Docker Desktop           │   │
│  │  (MINGW64)   │  │  ┌────────────────────────┐  │   │
│  │              │  │  │   Docker Engine         │  │   │
│  │  Terminal    │  │  │                         │  │   │
│  │  Operations  │  │  │  ┌──────┐  ┌──────┐   │  │   │
│  │              │  │  │  │ web  │  │redis │   │  │   │
│  │              │  │  │  │:80   │  │:6379 │   │  │   │
│  │              │  │  │  └──┬───┘  └──┬───┘   │  │   │
│  │              │  │  │     │         │        │  │   │
│  │              │  │  │  codyssey-net (bridge)  │  │   │
│  └─────────────┘  │  └────────────────────────┘  │   │
│                    └──────────────────────────────┘   │
│                                                       │
│  Port Mapping:  8080 ──→ web:80                       │
│                 8081 ──→ web:80 (2nd instance)        │
│                 6379 ──→ redis:6379                    │
│                                                       │
│  Volume:  ./site ──bind──→ /usr/share/nginx/html     │
│           redis-data ──vol──→ /data                   │
│                                                       │
│  Git Remote: github.com/dolphin1404/Codyssey          │
└─────────────────────────────────────────────────────┘
```

---

## 3. Docker 이미지와 컨테이너의 구조적 분리

### 3.1 레이어 구조

```
┌─────────────────────────┐
│   컨테이너 쓰기 레이어     │  ← docker run 시 생성, 삭제 시 소멸
├─────────────────────────┤
│   COPY site/ ...         │  ← Dockerfile의 각 명령이
├─────────────────────────┤     하나의 읽기 전용 레이어를 생성
│   ENV APP_ENV=dev        │
├─────────────────────────┤
│   nginx:alpine 베이스     │  ← FROM으로 지정한 베이스 이미지
└─────────────────────────┘
```

### 3.2 빌드 / 실행 / 변경 관점 비교

| 관점 | 이미지 | 컨테이너 |
|------|--------|----------|
| **빌드 시점** | Dockerfile → `docker build` → 이미지 생성 | 이미지 → `docker run` → 컨테이너 생성 |
| **실행 시점** | 실행 불가 (정적 파일 묶음) | 프로세스가 동작하는 격리 환경 |
| **변경 시점** | 변경 불가 (Immutable) | 쓰기 레이어에 변경 기록 (컨테이너 삭제 시 소멸) |
| **공유 방식** | Docker Hub / Registry로 push | 공유 불가 (호스트에 종속) |

### 3.3 설계 시 고려사항

- **이미지는 불변(Immutable)으로 유지**: 설정 변경이 필요하면 Dockerfile을 수정하고 새 이미지를 빌드
- **컨테이너는 일회용(Disposable)으로 취급**: 언제든 삭제하고 다시 생성할 수 있어야 함
- **영속 데이터는 볼륨으로 분리**: 컨테이너 삭제와 무관하게 데이터 유지

---

## 4. 포트 매핑 설계

### 4.1 포트 할당 규칙

```
호스트 포트 범위:
  8080~8089  →  웹 서비스 (HTTP)
  6379       →  Redis (기본 포트 유지)
  3306       →  MySQL (사용 시)
```

| 서비스 | 호스트 포트 | 컨테이너 포트 | 비고 |
|--------|-------------|---------------|------|
| web (1번) | 8080 | 80 | 기본 인스턴스 |
| web (2번) | 8081 | 80 | 테스트/비교용 |
| redis | 6379 | 6379 | 기본 포트 유지 |

### 4.2 포트 매핑이 필요한 이유

```
컨테이너 격리 구조:

  [호스트 네트워크]          [컨테이너 네트워크]
  localhost:8080   ──→   172.17.0.2:80 (web)
  localhost:8081   ──→   172.17.0.3:80 (web-2)
  localhost:6379   ──→   172.17.0.4:6379 (redis)

  * 컨테이너는 독립된 네트워크 네임스페이스를 가짐
  * -p 옵션 없이는 호스트에서 컨테이너 내부 포트에 접근 불가
  * 같은 이미지라도 호스트 포트만 다르면 여러 인스턴스 동시 실행 가능
```

### 4.3 포트 충돌 진단 절차

```
1. 오류 확인
   "Bind for 0.0.0.0:8080 failed: port is already allocated"

2. 포트 점유 프로세스 확인
   Windows:  netstat -ano | findstr :8080
   Linux:    lsof -i :8080 또는 ss -tlnp | grep :8080

3. 원인별 조치
   ├── Docker 컨테이너가 원인 → docker stop <name>
   ├── 다른 프로그램이 원인 → 해당 프로그램 종료 또는 포트 변경
   └── 원인 불명 → 다른 호스트 포트 사용 (예: 8082:80)
```

---

## 5. 스토리지 설계

### 5.1 바인드 마운트 vs Named 볼륨

```
바인드 마운트 (개발 시 사용):
  호스트 파일 ←→ 컨테이너 파일 (양방향 동기화)
  ./site/index.html ──→ /usr/share/nginx/html/index.html
  * 호스트에서 수정 → 컨테이너에 즉시 반영 (재시작 불필요)
  * 개발 중 실시간 변경 확인에 적합

Named 볼륨 (데이터 영속성):
  Docker 관리 볼륨 ──→ 컨테이너 마운트 포인트
  redis-data ──→ /data
  * 컨테이너 삭제해도 볼륨 데이터 유지
  * docker volume inspect로 실제 저장 위치 확인 가능
```

### 5.2 볼륨 네이밍 컨벤션

| 패턴 | 예시 | 용도 |
|------|------|------|
| `{서비스}-data` | `redis-data` | 서비스별 영속 데이터 |
| `{프로젝트}_{서비스}-data` | `codyssey_redis-data` | Compose 자동 생성 (프로젝트명 접두사) |

### 5.3 볼륨 생명주기

```
docker volume create mydata      # 생성
docker run -v mydata:/data ...   # 컨테이너에 연결
docker rm -f <container>         # 컨테이너 삭제 → 볼륨은 유지됨
docker run -v mydata:/data ...   # 새 컨테이너에서 기존 데이터 접근 가능
docker volume rm mydata          # 볼륨 명시적 삭제 시에만 데이터 소멸
```

---

## 6. 네트워크 설계

### 6.1 Compose 네트워크

```
codyssey-net (bridge):
  ┌──────────┐     ┌──────────┐
  │   web    │────→│  redis   │
  │          │     │          │
  │ DNS: web │     │DNS: redis│
  └──────────┘     └──────────┘

  * 같은 네트워크의 컨테이너는 서비스 이름으로 통신
  * web에서 redis-cli -h redis ping → PONG
  * 외부 네트워크와 격리되어 보안 확보
```

### 6.2 서비스 디스커버리

Docker Compose는 내장 DNS를 제공합니다:
- `redis`라는 서비스 이름이 자동으로 해당 컨테이너의 IP로 해석
- 컨테이너 IP가 변경되어도 서비스 이름은 동일하게 유지
- IP를 하드코딩할 필요 없음

---

## 7. 경로 전략

### 7.1 절대 경로 vs 상대 경로

```
절대 경로 (Absolute Path):
  /c/WorkSpace_2026/Codyssey/site/index.html
  * 루트(/)부터 시작하는 완전한 경로
  * 실행 위치에 관계없이 항상 같은 파일을 가리킴
  * Docker 볼륨 마운트, 시스템 설정 파일에서 사용

상대 경로 (Relative Path):
  ./site/index.html
  ../Codyssey/Dockerfile
  * 현재 디렉토리(.)를 기준으로 표현
  * 프로젝트 폴더를 통째로 옮겨도 유효
  * Dockerfile COPY, docker-compose.yml 볼륨에서 사용
```

### 7.2 프로젝트 내 경로 사용 규칙

| 파일 | 경로 방식 | 이유 |
|------|-----------|------|
| `Dockerfile` | 상대 (`COPY site/ ...`) | 빌드 컨텍스트 기준 |
| `docker-compose.yml` | 상대 (`./site:...`) | 프로젝트 이식성 |
| `docker run -v` | 절대 (`$(pwd)/site:...`) | Docker 데몬이 절대 경로를 요구 |
| 컨테이너 내부 | 절대 (`/usr/share/nginx/html`) | 컨테이너 파일시스템의 고정 위치 |

---

## 8. 재현성 보장 설계

### 8.1 재현 가능한 환경의 3가지 원칙

| 원칙 | 구현 방법 |
|------|-----------|
| **코드화 (Infrastructure as Code)** | Dockerfile, docker-compose.yml로 환경을 선언적으로 정의 |
| **버전 관리** | Git으로 설정 파일 변경 이력 추적 |
| **격리** | Docker 컨테이너로 호스트 환경과 분리 |

### 8.2 재현 절차

```bash
# 1. 저장소 클론
$ git clone https://github.com/dolphin1404/Codyssey.git
$ cd Codyssey

# 2. 전체 서비스 실행 (포트/볼륨/네트워크 모두 자동 설정)
$ docker compose up -d

# 3. 접속 확인
$ curl http://localhost:8080

# 4. 종료
$ docker compose down -v
```

### 8.3 환경별 차이 대응

| 차이점 | Windows (Git Bash) | Linux/Mac |
|--------|-------------------|-----------|
| 경로 변환 | `MSYS_NO_PATHCONV=1` 필요 | 불필요 |
| chmod | NTFS에서 미동작 → 컨테이너 내 실습 | 정상 동작 |
| Docker 설치 | Docker Desktop | Docker Engine (apt/brew) |
| 줄바꿈 | CRLF (Git이 LF로 자동 변환) | LF |

---

## 9. 환경 변수 설계 (보너스)

### 9.1 설정과 코드의 분리

```yaml
# docker-compose.yml
services:
  web:
    environment:
      - APP_ENV=dev          # 실행 환경 구분
      - NGINX_HOST=localhost # 호스트명 설정
```

| 원칙 | 설명 |
|------|------|
| 코드에 설정을 하드코딩하지 않음 | 환경(dev/staging/prod)마다 다른 값을 주입 |
| 환경 변수로 동작 변경 | 코드 수정 없이 설정만 바꿔서 재배포 가능 |
| `.env` 파일은 Git에 포함하지 않음 | 민감 정보(DB 비밀번호 등) 보호 |
