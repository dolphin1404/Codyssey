# AI/SW 개발 워크스테이션 구축

## 1. 프로젝트 개요

터미널(CLI), Docker(컨테이너), Git 설정을 직접 점검하고 실습한 기록입니다.
아래 내용은 실제 실행 로그 기준으로 정리했습니다.

## 2. 실행 환경

| 항목 | 버전/정보 |
|------|-----------|
| OS | macOS (OrbStack 사용) |
| Shell | zsh |
| Docker | 28.5.2 |
| Docker Compose | v2.40.3 |
| Docker Context | orbstack |

## 3. 수행 항목 체크리스트

- [x] 터미널 기본 조작 및 폴더 구성
- [x] 파일 생성/복사/이동/삭제 실습
- [x] Docker 설치/동작 점검
- [x] hello-world 컨테이너 실행
- [x] ubuntu 컨테이너 명령 실행
- [x] Dockerfile 기반 이미지 빌드
- [x] 포트 매핑으로 2개 웹 컨테이너 실행 (8080, 8081)
- [x] Docker 볼륨 영속성 검증
- [x] Git 전역 설정
- [x] Docker Compose 실행 시도

> 설계 문서: [docs/DESIGN.md](docs/DESIGN.md)

---

## 4. 터미널 기본 조작 로그
개인 정보로 usr_id 로 표시함

```bash
$ usr_id@c3r8s6 ~ % pwd
/Users/usr_id

$ usr_id@c3r8s6 ~ % ls
Desktop		Downloads	Movies		OrbStack	Public
Documents	Library		Music		Pictures

$ usr_id@c3r8s6 ~ % cd Downloads
$ usr_id@c3r8s6 Downloads % ls
Codyssey-main

$ usr_id@c3r8s6 Downloads % cd Codyssey-main 
$ usr_id@c3r8s6 Codyssey-main % ls
Dockerfile		docker-compose.yml	site
README.md		practice
$ usr_id@c3r8s6 Codyssey-main % pwd
/Users/usr_id/Downloads/Codyssey-main

usr_id@c3r8s6 Codyssey-main % ls -la
total 56
drwxr-xr-x@ 9 usr_id  usr_id    288 Mar 30 01:49 .
drwx------+ 5 usr_id  usr_id    160 Mar 30 17:51 ..
drwxr-xr-x@ 3 usr_id  usr_id     96 Mar 30 01:49 .claude
-rw-r--r--@ 1 usr_id  usr_id     45 Mar 30 01:49 .gitignore
-rw-r--r--@ 1 usr_id  usr_id    347 Mar 30 01:49 Dockerfile
-rw-r--r--@ 1 usr_id  usr_id  15266 Mar 30 01:49 README.md
-rw-r--r--@ 1 usr_id  usr_id    448 Mar 30 01:49 docker-compose.yml
drwxr-xr-x@ 3 usr_id  usr_id     96 Mar 30 01:49 site

usr_id@c3r8s6 Codyssey-main % mkdir -p practice
usr_id@c3r8s6 Codyssey-main % touch practice/hello.txt
usr_id@c3r8s6 Codyssey-main % echo "Hello Codyssey" > practice/hello.txt
usr_id@c3r8s6 Codyssey-main % cat practice/hello.txt
Hello Codyssey
usr_id@c3r8s6 Codyssey-main % cp practice/hello.txt practice/hello_backup.txt
usr_id@c3r8s6 Codyssey-main % mv practice/hello_backup.txt practice/renamed.txt
usr_id@c3r8s6 Codyssey-main % ls -la practice/
total 16
drwxr-xr-x@ 4 usr_id  usr_id  128 Mar 30 17:53 .
drwxr-xr-x@ 9 usr_id  usr_id  288 Mar 30 01:49 ..
-rw-r--r--@ 1 usr_id  usr_id   15 Mar 30 17:52 hello.txt
-rw-r--r--@ 1 usr_id  usr_id   15 Mar 30 17:53 renamed.txt

usr_id@c3r8s6 Codyssey-main % mv practice/hello_backup.txt practice/renamed.txt
mv: practice/hello_backup.txt: No such file or directory
usr_id@c3r8s6 Codyssey-main % ls -la practice 
total 16
drwxr-xr-x@ 4 usr_id  usr_id  128 Mar 30 17:53 .
drwxr-xr-x@ 9 usr_id  usr_id  288 Mar 30 01:49 ..
-rw-r--r--@ 1 usr_id  usr_id   15 Mar 30 17:52 hello.txt
-rw-r--r--@ 1 usr_id  usr_id   15 Mar 30 17:53 renamed.txt

usr_id@c3r8s6 Codyssey-main % rm practice/renamed.txt
usr_id@c3r8s6 Codyssey-main % ls -la practice        
total 8
drwxr-xr-x@ 3 usr_id  usr_id   96 Mar 30 17:54 .
drwxr-xr-x@ 9 usr_id  usr_id  288 Mar 30 01:49 ..
-rw-r--r--@ 1 usr_id  usr_id   15 Mar 30 17:52 hello.txt
```

---

## 5. 권한 실습 (컨테이너 내부)

권한 실습은 Ubuntu 컨테이너에서 시도했습니다.

```bash
$ docker run --rm ubuntu bash -c "
touch /tmp/test.txt && echo 'Hello' > /tmp/test.txt
echo '[before change] default permission:'
ls -l /tmp/test.txt
"
[before change] default permission:
-rw-r--r-- 1 root root 6 Mar 30 08:56 /tmp/test.txt
```

이후 아래처럼 명령 앞에 `$`를 붙여서 입력해 실행이 실패했습니다.

```bash
$ $ docker run --rm ubuntu bash -c "..."
zsh: command not found: $
```

권한 숫자 의미:
- `755` = 소유자 `rwx`, 그룹 `r-x`, 기타 `r-x`
- `644` = 소유자 `rw-`, 그룹 `r--`, 기타 `r--`
- `700` = 소유자 `rwx`, 그룹 `---`, 기타 `---`

---

## 6. Docker 설치 및 기본 점검

### 6.1 버전/상태 확인

```bash
$ docker --version
Docker version 28.5.2, build ecc6942

$ docker info
Client:
 Version:    28.5.2
 Context:    orbstack
 Plugins:
  buildx ... v0.29.1
  compose ... v2.40.3

Server:
 Containers: 0
 Running: 0
 Images: 1
 Operating System: OrbStack
 OSType: linux
 Architecture: x86_64
```

### 6.2 hello-world 실행

```bash
$ docker run hello-world
Unable to find image 'hello-world:latest' locally
...
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

### 6.3 ubuntu 컨테이너 명령 실행

```bash
$ docker run --rm ubuntu bash -c "echo 'Hello from Ubuntu container' && ls / && cat /etc/os-release | head -3"
Hello from Ubuntu container
bin
boot
...
PRETTY_NAME="Ubuntu 24.04.4 LTS"
NAME="Ubuntu"
VERSION_ID="24.04"
```

### 6.4 이미지/컨테이너 조회 및 로그 확인

```bash
$ docker images
REPOSITORY    TAG       IMAGE ID       CREATED       SIZE
hello-world   latest    e2ac70e7319a   6 days ago    10.1kB
ubuntu        latest    f794f40ddfff   4 weeks ago   78.1MB

$ docker ps -a
CONTAINER ID   IMAGE         COMMAND    STATUS                     NAMES
d1cfc8efbd54   hello-world   "/hello"   Exited (0) ...             busy_lichterman

$ docker logs d1cfc83fbd54
Error response from daemon: No such container: d1cfc83fbd54

$ docker logs docker-compose.yml
Error response from daemon: No such container: docker-compose.yml

$ docker logs d1
Hello from Docker!
...
```

핵심 확인:
- 컨테이너 ID를 잘못 입력하면 `No such container` 발생
- 파일명(`docker-compose.yml`)은 컨테이너 이름이 아니므로 `docker logs` 대상이 될 수 없음

---

## 7. Dockerfile 기반 커스텀 이미지

### 7.1 Dockerfile

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

### 7.2 빌드

```bash
$ docker build -t my-web:1.0 .
[+] Building ... FINISHED
=> [2/2] COPY site/ /usr/share/nginx/html/
=> naming to docker.io/library/my-web:1.0
```

---

## 8. 포트 매핑 컨테이너 실행

```bash
$ docker run -d -p 8080:80 --name my-web-8080 my-web:1.0
4bddf953f574859d6b94edf9e84f6caee2770dc5be2b3d3055c489e7d40ca65c

$ docker run -d -p 8081:80 --name my-web-8081 my-web:1.0
3fe323cc93fcb567c28538094b01dabbfc121eb23e08376ddd862ae9bd71a3b3

$ docker ps
CONTAINER ID   IMAGE        COMMAND                  STATUS                             PORTS                                     NAMES
3fe323cc93fc   my-web:1.0   "/docker-entrypoint.…"   Up ... (health: starting)          0.0.0.0:8081->80/tcp, [::]:8081->80/tcp   my-web-8081
4bddf953f574   my-web:1.0   "/docker-entrypoint.…"   Up ... (health: starting)          0.0.0.0:8080->80/tcp, [::]:8080->80/tcp   my-web-8080
```

핵심 확인:
- 같은 이미지 `my-web:1.0`으로 컨테이너 2개를 다른 호스트 포트로 동시에 실행

---

## 9. Docker 볼륨 영속성

```bash
$ docker volume create mydata
mydata

$ docker run -d --name vol-test -v mydata:/data ubuntu sleep infinity
bb5e73b6559d45a84da802e843aef06d09e1aa8d56a26c1cecf7dce3fd75f372

$ docker exec vol-test bash -c "echo 'Hello from Codyssey!' > /data/hello.txt && cat /data/hello.txt"
Hello from Codyssey!

$ docker rm -f vol-test
vol-test

$ docker run --rm -v mydata:/data ubuntu cat /data/hello.txt
Hello from Codyssey!
```

핵심 확인:
- 컨테이너를 삭제해도 named volume(`mydata`) 데이터는 유지됨

---

## 10. Git 설정

```bash
$ git config
error: no action specified

$ git config --global user.name KyuminLee
$ git config --global user.email "usr_id@gmail.com"
$ git config --global init.defaultBranch main

$ git config --list | grep -E "user.|init."
user.name=KyuminLee
user.email=usr_id@gmail.com
init.defaultbranch=main
```

핵심 확인:
- `git config`는 옵션 없이 실행하면 에러가 발생함
- 사용자 이름/이메일/기본 브랜치 설정 완료

---

## 11. Docker Compose 실행 기록

### 11.1 compose 실행 시도

```bash
$ docker compose up -d
[+] Running ...
Error response from daemon: failed to set up container networking: ... Bind for 0.0.0.0:8080 failed: port is already allocated
```

### 11.2 상태/로그 확인

```bash
$ docker compose ps
NAME                    IMAGE          SERVICE   STATUS          PORTS
codyssey-main-redis-1   redis:alpine   redis     Up ...          0.0.0.0:6379->6379/tcp, [::]:6379->6379/tcp

$ docker compose logs --tail 5
redis-1  | ... Ready to accept connections tcp
redis-1  | ... WARNING: Redis does not require authentication ...
```

### 11.3 네트워크 확인 시도 실패

```bash
$ docker exec codyssey-web-1 sh -c "redis-cli -h redis ping"
Error response from daemon: No such container: codyssey-web-1
```

### 11.4 정리

```bash
$ docker compose down -v
```

핵심 확인:
- 8080 포트가 이미 사용 중이라 `web` 서비스 기동 실패
- `redis` 서비스만 정상 실행됨

---

## 12. 트러블슈팅

### 12.1 명령 앞에 `$`를 같이 입력한 경우

| 항목 | 내용 |
|------|------|
| 문제 | `zsh: command not found: $` |
| 원인 | 문서 예시의 프롬프트 기호 `$`까지 복사해서 실행함 |
| 해결 | 실제 실행 시에는 `$`를 제외하고 명령어만 입력 |

### 12.2 잘못된 컨테이너 ID/이름으로 logs 조회

| 항목 | 내용 |
|------|------|
| 문제 | `No such container` |
| 원인 | 오타 ID(`d1cfc83fbd54`) 또는 파일명(`docker-compose.yml`) 사용 |
| 해결 | `docker ps -a`로 정확한 ID/이름 확인 후 `docker logs` 실행 |

### 12.3 Docker Compose 포트 충돌

| 항목 | 내용 |
|------|------|
| 문제 | `Bind for 0.0.0.0:8080 failed: port is already allocated` |
| 원인 | 이미 실행 중인 컨테이너(`my-web-8080`)가 8080 포트를 점유 |
| 해결 | 기존 8080 사용 컨테이너 중지/삭제 후 재시도 또는 compose 포트 변경 |

---

## 13. 프로젝트 구조

```
Codyssey/
├── README.md
├── Dockerfile
├── docker-compose.yml
├── docs/
│   └── DESIGN.md
├── site/
│   └── index.html
├── practice/
│   └── hello.txt
└── screenshots/
```

## 14. 검증 방법 요약

| 항목 | 검증 명령 | 로그 기준 결과 |
|------|-----------|----------------|
| Docker 설치 확인 | `docker --version` | 성공 (28.5.2) |
| Docker 동작 확인 | `docker run hello-world` | 성공 |
| Ubuntu 실행 확인 | `docker run --rm ubuntu ...` | 성공 |
| 이미지 빌드 | `docker build -t my-web:1.0 .` | 성공 |
| 포트 매핑 2개 실행 | `docker run -d -p 8080:80 ...` + `-p 8081:80 ...` | 성공 |
| 볼륨 영속성 | `docker volume create` + 재마운트 후 `cat` | 성공 |
| Git 전역 설정 | `git config --global ...` | 성공 |
| Compose 전체 기동 | `docker compose up -d` | 부분 성공 (redis만 기동) |
| web↔redis 통신 테스트 | `docker exec codyssey-web-1 ...` | 실패 (web 미기동) |
