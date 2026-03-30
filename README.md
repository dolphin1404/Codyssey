# AI/SW 개발 워크스테이션 구축

## 1. 프로젝트 개요

터미널(CLI), Docker(컨테이너), Git/GitHub(버전 관리 및 협업)를 직접 세팅하고 검증하여,
재현 가능한 개발 환경을 구축하는 미션입니다.

## 2. 실행 환경

| 항목 | 버전/정보 |
|------|-----------|
| OS | Windows 11 Home (10.0.26200) |
| Shell | Bash (MINGW64/Git Bash) |
| Docker | 27.4.0 (Docker Desktop) |
| Docker Compose | v2.31.0 |
| Git | 2.40.0.windows.1 |

## 3. 수행 항목 체크리스트

- [x] 터미널 기본 조작 및 폴더 구성
- [x] 권한 변경 실습 (파일 + 디렉토리)
- [x] Docker 설치/점검
- [x] hello-world 컨테이너 실행
- [x] ubuntu 컨테이너 진입 및 명령 수행
- [x] Dockerfile 기반 커스텀 이미지 빌드
- [x] 포트 매핑 접속 (8080, 8081 두 포트)
- [x] 바인드 마운트 반영 확인
- [x] Docker 볼륨 영속성 검증
- [x] Git 설정 + GitHub 연동
- [x] (보너스) Docker Compose 멀티 컨테이너
- [x] (보너스) 컨테이너 간 네트워크 통신 확인
- [x] (보너스) 환경 변수 활용

---

## 4. 터미널 조작 로그

### 4.1 기본 명령어

```bash
$ pwd
/c/WorkSpace_2026/Codyssey

$ ls -la
total 11
drwxr-xr-x 1 kyumi 197609   0 Mar 30 17:30 .
drwxr-xr-x 1 kyumi 197609   0 Mar 30 17:26 ..
-rw-r--r-- 1 kyumi 197609  45 Mar 30 17:30 .gitignore
-rw-r--r-- 1 kyumi 197609 347 Mar 30 17:30 Dockerfile
-rw-r--r-- 1 kyumi 197609 464 Mar 30 17:30 docker-compose.yml
drwxr-xr-x 1 kyumi 197609   0 Mar 30 17:30 screenshots
drwxr-xr-x 1 kyumi 197609   0 Mar 30 17:30 site

$ mkdir -p practice
$ ls -la
# practice 디렉토리가 추가된 것을 확인

$ touch practice/hello.txt
$ echo "Hello Codyssey" > practice/hello.txt
$ cat practice/hello.txt
Hello Codyssey

$ cp practice/hello.txt practice/hello_backup.txt
$ ls -la practice/
-rw-r--r-- 1 kyumi 197609 15 Mar 30 17:31 hello.txt
-rw-r--r-- 1 kyumi 197609 15 Mar 30 17:31 hello_backup.txt

$ mv practice/hello_backup.txt practice/renamed.txt
$ ls -la practice/
-rw-r--r-- 1 kyumi 197609 15 Mar 30 17:31 hello.txt
-rw-r--r-- 1 kyumi 197609 15 Mar 30 17:31 renamed.txt

$ rm practice/renamed.txt
$ ls -la practice/
-rw-r--r-- 1 kyumi 197609 15 Mar 30 17:31 hello.txt
```

### 4.2 권한 실습

> **참고:** Windows(NTFS)에서는 `chmod`가 실제 동작하지 않으므로, Docker 컨테이너(Linux) 내에서 수행했습니다.

```bash
$ docker run --rm ubuntu bash -c "
  touch /tmp/test.txt && echo 'Hello' > /tmp/test.txt

  echo '[변경 전] 기본 권한:'
  ls -l /tmp/test.txt
  # -rw-r--r-- (644)

  chmod 755 /tmp/test.txt
  echo '[chmod 755 후] 소유자rwx, 그룹rx, 기타rx:'
  ls -l /tmp/test.txt
  # -rwxr-xr-x (755)

  chmod 644 /tmp/test.txt
  echo '[chmod 644 후] 소유자rw, 그룹r, 기타r:'
  ls -l /tmp/test.txt
  # -rw-r--r-- (644)

  mkdir /tmp/testdir
  echo '[디렉토리 변경 전]:'
  ls -ld /tmp/testdir
  # drwxr-xr-x (755)

  chmod 700 /tmp/testdir
  echo '[chmod 700 후] 소유자만 rwx:'
  ls -ld /tmp/testdir
  # drwx------ (700)

  chmod 755 /tmp/testdir
  echo '[chmod 755 후]:'
  ls -ld /tmp/testdir
  # drwxr-xr-x (755)
"
```

**권한 해석:**
| 숫자 | 의미 | rwx 표기 |
|------|------|----------|
| 7 | 읽기+쓰기+실행 | rwx |
| 5 | 읽기+실행 | r-x |
| 4 | 읽기만 | r-- |
| 6 | 읽기+쓰기 | rw- |
| 0 | 없음 | --- |

- `755` = 소유자(rwx) + 그룹(r-x) + 기타(r-x) → 실행 파일이나 디렉토리에 주로 사용
- `644` = 소유자(rw-) + 그룹(r--) + 기타(r--) → 일반 파일에 주로 사용
- `700` = 소유자(rwx) + 그룹(---) + 기타(---) → 소유자만 접근 가능

---

## 5. Docker 설치 및 기본 점검

### 5.1 버전 확인

```bash
$ docker --version
Docker version 27.4.0, build bde2b89

$ docker info
Client:
 Version:    27.4.0
 Context:    desktop-linux
 Plugins:
  compose: Docker Compose (v2.31.0-desktop.2)
  buildx: Docker Buildx (v0.19.2-desktop.1)
Server:
 Containers: 0
 Running: 0
 ...
```

### 5.2 hello-world 실행

```bash
$ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
...
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

### 5.3 Ubuntu 컨테이너 진입

```bash
$ docker run --rm ubuntu bash -c "echo 'Hello from Ubuntu container!' && ls / && cat /etc/os-release | head -3"
Hello from Ubuntu container!
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
PRETTY_NAME="Ubuntu 24.04.1 LTS"
NAME="Ubuntu"
VERSION_ID="24.04"
```

### 5.4 이미지/컨테이너 관리 명령

```bash
$ docker images
REPOSITORY    TAG       IMAGE ID       CREATED         SIZE
hello-world   latest    452a468a4bf9   6 days ago      20.4kB
ubuntu        latest    80dd3c3b9c6c   16 months ago   117MB

$ docker ps -a
CONTAINER ID   IMAGE         COMMAND    CREATED         STATUS                     PORTS   NAMES
c6a3993e1692   hello-world   "/hello"   1 minute ago    Exited (0) 1 minute ago            boring_leavitt

$ docker logs c6a3993e1692
Hello from Docker!
This message shows that your installation appears to be working correctly.
...

$ docker stats --no-stream
CONTAINER ID   NAME          CPU %   MEM USAGE / LIMIT     MEM %   NET I/O          BLOCK I/O   PIDS
8ce3bb8f820f   my-web-8081   0.00%   13.11MiB / 7.608GiB   0.17%  1.42kB / 1.8kB   0B / 0B     17
4fe122604e77   my-web-8080   0.00%   13.11MiB / 7.608GiB   0.17%  1.72kB / 1.8kB   0B / 0B     17
```

**attach vs exec 차이:**
| 명령 | 동작 | 종료 시 |
|------|------|---------|
| `docker attach` | 컨테이너의 메인 프로세스(PID 1)에 연결 | `exit` 시 컨테이너도 종료됨 |
| `docker exec` | 컨테이너 내부에 새 프로세스를 생성하여 실행 | `exit` 시 해당 프로세스만 종료, 컨테이너는 계속 실행 |

---

## 6. Dockerfile 기반 커스텀 이미지

### 6.1 베이스 이미지 및 커스텀 포인트

- **베이스:** `nginx:alpine` (경량 웹 서버)
- **커스텀 포인트:**
  - `LABEL`: 이미지 메타데이터 추가 (제목, 설명)
  - `ENV APP_ENV=dev`: 환경 변수 주입
  - `COPY site/`: 정적 HTML 콘텐츠 배포
  - `HEALTHCHECK`: 30초 간격으로 서버 상태 자동 점검

### 6.2 Dockerfile

```dockerfile
FROM nginx:alpine

LABEL maintainer="codyssey"
LABEL org.opencontainers.image.title="codyssey-web"
LABEL org.opencontainers.image.description="Codyssey Dev Workstation custom nginx image"

ENV APP_ENV=dev

COPY site/ /usr/share/nginx/html/

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=3s \
    CMD wget -q --spider http://localhost/ || exit 1
```

### 6.3 빌드 및 실행

```bash
$ docker build -t my-web:1.0 .
#6 [2/2] COPY site/ /usr/share/nginx/html/
#7 exporting to image
#7 naming to docker.io/library/my-web:1.0
```

---

## 7. 포트 매핑 접속 증거

### 7.1 두 개 포트로 동시 실행

```bash
$ docker run -d -p 8080:80 --name my-web-8080 my-web:1.0
4fe122604e77...

$ docker run -d -p 8081:80 --name my-web-8081 my-web:1.0
8ce3bb8f820f...

$ docker ps
CONTAINER ID   IMAGE        COMMAND                   STATUS                   PORTS
8ce3bb8f820f   my-web:1.0   "/docker-entrypoint.…"   Up (healthy)             0.0.0.0:8081->80/tcp
4fe122604e77   my-web:1.0   "/docker-entrypoint.…"   Up (healthy)             0.0.0.0:8080->80/tcp
```

### 7.2 접속 확인 (curl)

```bash
$ curl http://localhost:8080 | head -5
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>Codyssey - AI/SW 개발 워크스테이션</title>

$ curl http://localhost:8081 | head -5
<!DOCTYPE html>
<html lang="ko">
...
```

> 브라우저에서 `http://localhost:8080`, `http://localhost:8081` 접속 시 동일한 페이지가 표시됩니다.

**포트 매핑이 필요한 이유:**
컨테이너는 격리된 네트워크 환경에서 실행됩니다. 컨테이너 내부의 포트(80)는 호스트에서 직접 접근할 수 없으며,
`-p <호스트포트>:<컨테이너포트>`로 매핑해야 호스트에서 접속할 수 있습니다.
같은 이미지라도 호스트 포트를 다르게 지정하면 여러 인스턴스를 동시에 실행할 수 있습니다.

---

## 8. 바인드 마운트 + 볼륨 영속성

### 8.1 바인드 마운트 (호스트 파일 변경 → 컨테이너 즉시 반영)

```bash
# 바인드 마운트로 실행
$ docker run -d -p 8080:80 --name bind-test \
    -v "$(pwd)/site:/usr/share/nginx/html" nginx:alpine

# 변경 전 확인
$ curl -s http://localhost:8080 | grep "<h1>"
    <h1>Codyssey Dev Workstation</h1>

# 호스트에서 index.html 수정 (제목 변경)
$ sed -i 's/Codyssey Dev Workstation/Codyssey Dev Workstation - Updated via Bind Mount!/' site/index.html

# 변경 후 확인 (컨테이너 재시작 없이 즉시 반영!)
$ curl -s http://localhost:8080 | grep "<h1>"
    <h1>Codyssey Dev Workstation - Updated via Bind Mount!</h1>
```

### 8.2 Docker 볼륨 영속성 (컨테이너 삭제 후에도 데이터 유지)

```bash
# 1. 볼륨 생성
$ docker volume create mydata
mydata

# 2. 컨테이너에서 볼륨에 데이터 쓰기
$ docker run -d --name vol-test -v mydata:/data ubuntu sleep infinity
$ docker exec vol-test bash -c "echo 'Hello from Codyssey!' > /data/hello.txt && cat /data/hello.txt"
Hello from Codyssey!

# 3. 컨테이너 삭제
$ docker rm -f vol-test
vol-test

# 4. 새 컨테이너에서 데이터 확인 → 유지됨!
$ MSYS_NO_PATHCONV=1 docker run --rm -v mydata:/data ubuntu cat /data/hello.txt
Hello from Codyssey!
```

**Docker 볼륨이란:**
Docker 볼륨은 컨테이너의 라이프사이클과 독립적으로 데이터를 저장하는 메커니즘입니다.
컨테이너를 삭제해도 볼륨에 저장된 데이터는 유지되며, 다른 컨테이너에 다시 연결할 수 있습니다.
데이터베이스, 로그, 설정 파일 등 영속성이 필요한 데이터에 사용합니다.

---

## 9. Git 설정 및 GitHub 연동

### 9.1 Git 설정

```bash
$ git config --global user.name "KyuminLee"
$ git config --global user.email "kyumin1404@gmail.com"
$ git config --global init.defaultBranch main

$ git config --list | grep -E "user.|init."
user.name=KyuminLee
user.email=kyumin1404@gmail.com
init.defaultbranch=main
```

### 9.2 저장소 초기화 및 GitHub 연동

```bash
$ git init
$ git remote add origin https://github.com/dolphin1404/Codyssey.git
$ git add .
$ git commit -m "Initial commit: 개발 워크스테이션 구축"
$ git push -u origin main
```

**Git과 GitHub의 역할 차이:**
| 구분 | Git | GitHub |
|------|-----|--------|
| 역할 | 로컬 버전 관리 도구 | 원격 협업 플랫폼 |
| 위치 | 내 컴퓨터 | 클라우드 서버 |
| 기능 | 커밋, 브랜치, 머지 등 변경 이력 추적 | 코드 호스팅, PR, 이슈, CI/CD 등 협업 기능 |
| 오프라인 | 사용 가능 | 인터넷 필요 |

---

## 10. (보너스) Docker Compose

### 10.1 docker-compose.yml

```yaml
services:
  web:
    build: .
    ports:
      - "8080:80"
    environment:
      - APP_ENV=dev
      - NGINX_HOST=localhost
    volumes:
      - ./site:/usr/share/nginx/html
    depends_on:
      - redis
    networks:
      - codyssey-net

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - codyssey-net

volumes:
  redis-data:

networks:
  codyssey-net:
    driver: bridge
```

### 10.2 실행 및 확인

```bash
# 실행
$ docker compose up -d
 Container codyssey-redis-1  Created
 Container codyssey-web-1    Created
 Container codyssey-redis-1  Started
 Container codyssey-web-1    Started

# 상태 확인
$ docker compose ps
NAME               IMAGE          SERVICE   STATUS                    PORTS
codyssey-redis-1   redis:alpine   redis     Up                        0.0.0.0:6379->6379/tcp
codyssey-web-1     codyssey-web   web       Up (health: starting)     0.0.0.0:8080->80/tcp

# 로그 확인
$ docker compose logs --tail 5
redis-1  | Ready to accept connections tcp
web-1    | start worker process 41...

# 컨테이너 간 네트워크 통신 확인
$ docker exec codyssey-web-1 sh -c "redis-cli -h redis ping"
PONG

# 종료
$ docker compose down -v
```

**배운 점:**
- `docker compose up/down/ps/logs`로 여러 서비스를 한꺼번에 관리
- `depends_on`으로 서비스 실행 순서를 제어
- 같은 네트워크(`codyssey-net`) 내 컨테이너는 서비스 이름(`redis`)으로 서로 통신 가능
- 환경 변수(`APP_ENV`, `NGINX_HOST`)로 설정과 코드를 분리

---

## 11. 트러블슈팅

### 트러블슈팅 1: Windows에서 chmod가 동작하지 않음

| 항목 | 내용 |
|------|------|
| **문제** | Git Bash(MINGW64)에서 `chmod 755 hello.txt` 실행 후 `ls -l`을 확인하면 권한이 변경되지 않음 |
| **원인 가설** | Windows NTFS 파일시스템은 Unix 권한 모델(rwx)을 지원하지 않음 |
| **확인** | chmod 실행 전후 `ls -l` 결과가 동일함 (`-rw-r--r--` 유지) |
| **해결** | Docker 컨테이너(Ubuntu) 내부에서 권한 실습을 수행하여 Linux 환경에서 정상 동작 확인 |

### 트러블슈팅 2: MSYS 경로 변환 문제

| 항목 | 내용 |
|------|------|
| **문제** | `docker run --rm -v mydata:/data ubuntu cat /data/hello.txt` 실행 시 `cat: 'C:/Program Files/Git/data/hello.txt': No such file or directory` 오류 |
| **원인 가설** | Git Bash(MSYS)가 `/data/...` 경로를 Windows 경로로 자동 변환함 |
| **확인** | MSYS는 Unix 스타일 절대경로(`/data`)를 Windows 경로(`C:/Program Files/Git/data`)로 변환하는 동작이 있음 |
| **해결** | `MSYS_NO_PATHCONV=1` 환경변수를 명령 앞에 추가하여 경로 변환 비활성화 |

```bash
# 오류 발생
$ docker run --rm -v mydata:/data ubuntu cat /data/hello.txt
cat: 'C:/Program Files/Git/data/hello.txt': No such file or directory

# 해결
$ MSYS_NO_PATHCONV=1 docker run --rm -v mydata:/data ubuntu cat /data/hello.txt
Hello from Codyssey!
```

---

## 12. 프로젝트 구조

```
Codyssey/
├── README.md              # 기술 문서 (본 파일)
├── Dockerfile             # nginx:alpine 기반 커스텀 이미지
├── docker-compose.yml     # (보너스) 멀티 컨테이너 구성
├── .gitignore             # Git 제외 파일 목록
├── site/
│   └── index.html         # 정적 웹 페이지
├── practice/
│   └── hello.txt          # 터미널 실습 파일
└── screenshots/           # 스크린샷 증거
```

---

## 13. 검증 방법 요약

| 항목 | 검증 명령 | 기대 결과 |
|------|-----------|-----------|
| Docker 설치 | `docker --version` | 27.4.0 출력 |
| hello-world | `docker run hello-world` | "Hello from Docker!" 메시지 |
| 커스텀 빌드 | `docker build -t my-web:1.0 .` | 빌드 성공 |
| 포트 매핑 | `curl http://localhost:8080` | HTML 응답 |
| 바인드 마운트 | 호스트 파일 수정 후 curl | 변경 내용 반영 |
| 볼륨 영속성 | 컨테이너 삭제 후 새 컨테이너에서 cat | 데이터 유지 |
| Compose | `docker compose up -d && docker compose ps` | 2개 서비스 Running |
| 네트워크 | `docker exec web redis-cli -h redis ping` | PONG |
