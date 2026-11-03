# 내 컴퓨터에 개발자용 작업실 꾸미기

리눅스 CLI, Docker, Git/GitHub를 이용해 개발 워크스테이션을 구성한 실습 기록이다. NGINX 기반 커스텀 이미지를 만들고, 포트 매핑으로 웹 페이지에 접속한 뒤 바인드 마운트의 변경 반영과 Docker 볼륨의 데이터 영속성을 확인했다.

> 실습한 내용은 명령어와 출력으로 정리하고, 직접 촬영한 스크린샷과 저장된 로그를 함께 첨부했다. 포트 매핑 접속 화면에는 주소창과 응답 화면이 포함되어 있다. 재현용 명령과 실제 관찰 결과는 구분하여 기록했다.

## 1. 실행 환경

| 항목 | 실행 환경 |
| --- | --- |
| 호스트 OS | Windows, 빌드 26100 계열 |
| Linux 환경 | WSL 2 / Ubuntu 26.04.1 LTS |
| 쉘 / 터미널 | Windows PowerShell 5.1.26100.9711 및 WSL Linux 터미널 |
| Docker | CLI / Engine 29.9.2, Docker Desktop 4.97.0 |
| Git | 2.56.0.windows.4 |
| 에디터 | VS Code 1.144.1, x64 |
| 실습 컨테이너 | ubuntu:22.04, nginx:alpine |

원본 실습 경로는 Windows의 `C:\github\codyssey_E1-1`, WSL의 `/mnt/c/github/codyssey_E1-1`이다. 다른 PC에서는 자신의 저장소 경로로 바꾼다. 아래 Linux 명령은 특별한 표시가 없으면 WSL 터미널에서 실행한다. Docker Desktop 실행 및 해당 WSL 배포판 연동이 필요하다.

## 2. 수행 체크리스트

- [x] 현재 위치, 숨김 파일 포함 목록, 디렉토리 생성 및 이동
- [x] 파일 생성, 내용 확인, 복사, 이동/이름 변경, 삭제
- [x] 파일 1개와 디렉토리 1개의 권한 변경 전후 확인
- [x] Docker 버전 및 서버 응답 확인 (`docker version`)
- [x] 이미지 다운로드/목록, 컨테이너 실행 및 전체 목록 확인
- [x] hello-world 및 Ubuntu 컨테이너 내부 명령 실행
- [x] Dockerfile 기반 커스텀 이미지 빌드
- [x] 8080/8081 포트 매핑 및 웹 접속 확인
- [x] 바인드 마운트로 호스트 파일 변경 반영
- [x] 컨테이너 삭제 후 Docker 볼륨 데이터 유지
- [x] Git 사용자 정보, 원격 저장소 설정 및 GitHub 커밋 이력 확인
- [x] Git 기본 브랜치 설정 확인 (`master`; 현재 저장소는 `main`)

## 3. 저장소 구성

```text
codyssey_E1-1/
├── README.md
├── image/                         # 실습 스크린샷
└── developer-workstation/
    ├── Dockerfile
    ├── app/index.html             # NGINX로 제공할 정적 페이지
    ├── terminal/practice.txt      # 파일 조작 결과
    ├── permissions/permission-test.txt
    └── logs/
        ├── docker-practice.txt
        ├── permission-practice.txt
        ├── web-server-practice.txt
        └── storage-practice.txt
```

![실습 디렉토리 구조](image/tree구조.png)

빈 디렉토리는 Git에서 추적하지 않으므로, 권한 실습용 `test-directory`는 필요할 때 다시 생성한다.

## 4. 터미널 기본 조작

### 4.1 위치 확인과 디렉토리 구성

```bash
pwd
mkdir -p developer-workstation
cd developer-workstation
pwd
mkdir -p terminal permissions app screenshots logs
ls
ls -la
```

실습 출력 발췌:

```text
/mnt/c/github/codyssey_E1-1/developer-workstation
app  logs  permissions  screenshots  terminal
```

![디렉토리 생성 및 목록 확인](image/폴더생성및ls확인.png)

`pwd`는 현재 위치를, `ls -la`는 숨김 항목을 포함한 상세 목록을 보여준다.

절대 경로는 `/mnt/c/github/codyssey_E1-1/developer-workstation/app`처럼 루트(`/`)부터 지정한다. 상대 경로 `app/index.html`은 현재 디렉토리를 기준으로 해석한다. `cd ..`은 상위 디렉토리로 이동한다.

### 4.2 파일 생성·읽기·복사·이동·삭제

```bash
touch terminal/practice.txt
echo "hello linux!" > terminal/practice.txt
echo "i am learning docker and git." >> terminal/practice.txt
cat terminal/practice.txt
cp terminal/practice.txt terminal/practice-copy.txt
mv terminal/practice-copy.txt logs/
mv logs/practice-copy.txt logs/docker-practice.txt
touch terminal/delete-test.txt
rm terminal/delete-test.txt
```

`cat` 출력:

```text
hello linux!
i am learning docker and git.
```

`>`는 덮어쓰기, `>>`는 이어쓰기다. `cp`는 복사, `mv`는 이동 또는 이름 변경, `rm`은 삭제다. 성공한 `cp`, `mv`, `touch`, `rm`은 별도 출력 없이 프롬프트로 돌아왔다.

![파일 생성과 내용 확인](image/touch및echo.png)

![파일 복사 이동 삭제](<image/cp,mv,touch,rm.png>)

[파일 결과](developer-workstation/terminal/practice.txt), [복사된 로그 파일](developer-workstation/logs/docker-practice.txt)

## 5. 파일 및 디렉토리 권한

```bash
mkdir -p permissions/test-directory
touch permissions/permission-test.txt
ls -l permissions/permission-test.txt
ls -ld permissions/test-directory
chmod 644 permissions/permission-test.txt
ls -l permissions/permission-test.txt
chmod 755 permissions/permission-test.txt
ls -l permissions/permission-test.txt
chmod 644 permissions/permission-test.txt
chmod 755 permissions/test-directory
ls -ld permissions/test-directory
chmod 700 permissions/test-directory
ls -ld permissions/test-directory
chmod 755 permissions/test-directory
```

변경 전후 관찰:

```text
파일 644: -rw-r--r--
파일 755: -rwxr-xr-x
디렉토리 755: drwxr-xr-x
디렉토리 700: drwx------
```

![파일 및 디렉토리 권한 변경](<image/chmod 확인 화면.png>)

권한은 소유자 / 그룹 / 기타 사용자 순으로 읽는다. `r=4`(읽기), `w=2`(쓰기), `x=1`(실행)을 합산하므로 `755`는 `rwx/r-x/r-x`, `644`는 `rw-/r--/r--`다. 디렉토리의 `x`는 진입 및 하위 경로 접근 권한을 뜻한다. 텍스트 파일은 최종 `644`, 디렉토리는 `755`로 되돌렸다.

[최종 권한 기록](developer-workstation/logs/permission-practice.txt). WSL의 Windows 드라이브 경로에서는 마운트 설정에 따라 권한 동작이 달라질 수 있으므로, 같은 결과가 나오지 않으면 WSL 홈 디렉토리에서도 비교한다.

## 6. Docker 기본 점검과 컨테이너 실행

### 6.1 버전 및 데몬 응답

```bash
docker --version
docker version
```

출력 발췌:

```text
Docker version 29.9.2, build a82e528
Client: Version 29.9.2
Server: Docker Desktop 4.97.0 (244584)
Engine: Version 29.9.2
OS/Arch: linux/amd64
```

Client뿐 아니라 Server 정보도 출력되어 Docker 엔진에 연결됨을 확인했다.

![Docker 버전 및 서버 응답](<image/docker version.png>)

### 6.2 hello-world와 Ubuntu

```bash
docker run hello-world
docker run -it --name ubuntu-practice ubuntu:22.04 bash
# 아래는 Ubuntu 컨테이너 내부에서 실행
pwd
ls
echo "Hello from Ubuntu container"
cat /etc/os-release
exit
# 다시 호스트에서 실행
docker ps -a
```

![hello-world 실행 결과](<image/docker run hello-world 결과.png>)

![Ubuntu 컨테이너 내부 실습](<image/Ubuntu 컨테이너 내부 명령어 실행 결과.png>)

![전체 컨테이너 상태 확인](<image/docker ps -a 결과.png>)

Ubuntu 실습 출력 발췌:

```text
/
Hello from Ubuntu container
PRETTY_NAME="Ubuntu 22.04.5 LTS"
```

`docker run`은 이미지가 없으면 다운로드하고 새 컨테이너를 생성한다. Ubuntu의 주 프로세스인 `bash`에서 `exit`하자 `ubuntu-practice`가 `Exited (0)`으로 표시되었다. hello-world도 메시지를 출력하고 정상 종료했다. 컨테이너 종료와 이미지 삭제는 별개다.

`docker exec`는 실행 중인 컨테이너에 별도 프로세스를 실행한다. 이 실습에서는 볼륨의 파일을 읽는 데 사용했다. `docker attach`는 주 프로세스의 입출력에 연결한다.

## 7. Dockerfile 기반 웹 서버

### 7.1 베이스 이미지와 커스텀 포인트

NGINX와 Alpine Linux가 구성된 `nginx:alpine`을 베이스로 선택했다. [직접 작성한 Dockerfile](developer-workstation/Dockerfile)은 다음과 같다.

```dockerfile
FROM nginx:alpine

LABEL org.opencontainers.image.title="developer-workstation-web"

ENV APP_ENV=development

COPY app/ /usr/share/nginx/html/

EXPOSE 80
```

| 설정 | 목적 |
| --- | --- |
| `FROM nginx:alpine` | 기존 NGINX 실행 환경 재사용 |
| `LABEL` | 이미지의 프로젝트 이름 기록 |
| `ENV APP_ENV=development` | 개발 환경 값 정의. 현재 정적 HTML은 이 값을 읽지 않음 |
| `COPY app/ ...` | 기본 페이지 대신 직접 작성한 HTML 제공 |
| `EXPOSE 80` | 컨테이너 서비스 포트 문서화. 실제 호스트 공개는 `-p`로 설정 |

[웹 페이지 소스](developer-workstation/app/index.html)는 제목, 설명, 배경색, 카드 형태의 스타일을 포함한다. 현재 소스에는 바인드 마운트 실험 후의 `바인드테스트` 문구가 남아 있다.

### 7.2 빌드 및 실행

저장소 루트에서 시작한다. 이미 같은 이름의 컨테이너가 있으면 다른 이름과 사용하지 않는 호스트 포트를 지정한다.

```bash
cd developer-workstation
docker build --no-cache -t developer-workstation-web:1.0 .
docker images
docker run -d --name dev-web -p 8080:80 developer-workstation-web:1.0
docker ps
curl http://localhost:8080
```

실습 출력 발췌:

```text
[+] Building 1.3s (7/7) FINISHED
naming to docker.io/library/developer-workstation-web:1.0

IMAGE                            ID
developer-workstation-web:1.0    5a82bd0bdb55

STATUS          PORTS                                     NAMES
Up 21 minutes   0.0.0.0:8080->80/tcp, [::]:8080->80/tcp   dev-web
```

![커스텀 이미지 빌드 성공](<image/Docker 이미지 빌드 성공 화면.png>)

![웹 서버 컨테이너 실행 상태](<image/dev-web이 실행 중인 화면.png>)

[이미지 목록·실행 상태·HTML 응답 로그](developer-workstation/logs/web-server-practice.txt)

### 7.3 포트 매핑과 브라우저 접속

`-p 8080:80`은 호스트 8080번 포트의 요청을 컨테이너의 NGINX 80번 포트로 전달한다. 컨테이너는 자체 네트워크 환경을 사용하므로 호스트에서 접근할 포트를 명시적으로 연결한다.

주소창의 `localhost:8080`과 `Hello, Docker!` 응답을 확인했다.

![8080 웹 페이지 접속](<image/브라우저에서 직접 만든 웹 페이지가 보이는 화면.png>)

현재 소스로 다시 빌드하면 제목에 `바인드테스트`가 포함된다. 위 화면은 해당 문구를 추가하기 전 이미지로 실행한 결과다. `nginx:alpine`은 변경 가능한 태그이므로 재빌드 시 이미지 ID나 NGINX 버전은 달라질 수 있다.

## 8. 바인드 마운트: 호스트 변경 반영

작업 위치는 `developer-workstation`이다.

```bash
docker run -d \
  --name web-bind \
  -p 8081:80 \
  -v "$PWD/app:/usr/share/nginx/html:ro" \
  nginx:alpine
docker ps
curl -i http://localhost:8081
docker inspect web-bind --format '{{json .Mounts}}'
```

실습 응답 발췌:

```text
HTTP/1.1 200 OK
Server: nginx/1.31.8
Content-Type: text/html
```

![바인드 마운트 실행 및 HTTP 응답](<image/Bind Mount 컨테이너 실행 및 포트 확인.png>)

호스트 `app/index.html`의 제목을 다음과 같이 바꾸고 브라우저를 새로고침했다.

```html
<!-- 변경 전 -->
<h1>Hello, Docker!</h1>
<!-- 변경 후 -->
<h1>Hello, Docker! 바인드테스트</h1>
```

![8081에서 변경된 문구 확인](<image/Bind Mount 컨테이너.png>)

변경 전 HTML은 [기존 응답 로그](developer-workstation/logs/web-server-practice.txt)에 기록되어 있다. 변경 후 8081 브라우저 화면에는 `바인드테스트`가 표시된다. 변경 전후는 위 HTML 코드와 응답 기록으로 비교한다.

바인드 마운트는 호스트 디렉토리를 컨테이너에 연결하므로 이미지 재빌드 없이 수정 내용이 반영된다. `:ro`는 컨테이너 측 쓰기를 막으며 호스트에서의 편집은 가능하다. `COPY`로 이미지에 넣은 파일은 호스트 파일을 수정해도 자동으로 바뀌지 않는다.

[마운트 로그](developer-workstation/logs/storage-practice.txt)에서 `Type: bind`, `Destination: /usr/share/nginx/html`, `RW: false`를 확인했다.

## 9. Docker 볼륨: 컨테이너 삭제 후 데이터 유지

볼륨 생성, 데이터 저장, 컨테이너 삭제 및 재연결 절차는 다음과 같다. 삭제 후 데이터 확인 결과를 로그와 스크린샷으로 첨부했다.

```bash
docker volume create site-data
docker run -d --name volume-test -v site-data:/data ubuntu:22.04 sleep infinity
docker exec volume-test sh -c 'echo "This data will survive container deletion!" > /data/message.txt'
docker exec volume-test cat /data/message.txt
# 실습용 컨테이너만 삭제하며 site-data 볼륨은 유지한다.
docker rm -f volume-test
docker volume ls
docker run -d --name volume-test2 -v site-data:/data ubuntu:22.04 sleep infinity
docker exec volume-test2 cat /data/message.txt
```

삭제 후 관찰한 핵심 출력:

```text
$ docker rm -f volume-test
volume-test
$ docker volume ls
DRIVER    VOLUME NAME
local     site-data
$ docker exec volume-test2 cat /data/message.txt
This data will survive container deletion!
```

![컨테이너 삭제 및 볼륨 데이터 유지 확인](<image/컨테이너 삭제 후에도 데이터가 유지된 결과.png>)

[볼륨 및 데이터 로그](developer-workstation/logs/storage-practice.txt)

컨테이너를 삭제해도 별도로 관리되는 `site-data` 볼륨은 남았다. 새 컨테이너에 같은 볼륨을 `/data`로 연결하자 기존 메시지를 읽을 수 있었다. 바인드 마운트가 호스트의 특정 경로를 연결하는 방식이라면, 명명된 볼륨은 Docker가 저장 위치를 관리하는 방식이다.

### 9.1 데이터 손실 방지를 위한 볼륨 백업

볼륨은 컨테이너와 별도로 유지되지만, 볼륨 자체 삭제나 데이터 덮어쓰기까지 방지하지는 못한다. 따라서 삭제·수정 작업 전에 볼륨 내용을 호스트의 압축 파일로 백업하고, 복구 가능 여부를 확인한다. 아래는 `site-data` 볼륨의 `/data/message.txt`를 기준으로 한 백업·복구 예시다. 명령은 WSL의 `developer-workstation` 디렉토리에서 실행한다.

**1. 쓰기 작업 중단 및 백업 파일 생성**

`docker ps --filter volume=site-data`로 볼륨을 사용하는 컨테이너를 확인한다. 이 실습에서는 `volume-test2`를 중지한 후 백업한다. 다른 컨테이너도 같은 볼륨에 쓰고 있다면 함께 중지한다. 데이터베이스는 해당 DB의 덤프 또는 일관된 백업 기능을 사용한다.

```bash
docker ps --filter volume=site-data
docker stop volume-test2
mkdir -p backups
backup_name="site-data-$(date +%Y%m%d-%H%M%S).tar.gz"
docker run --rm \
  --mount type=volume,src=site-data,dst=/data,readonly \
  --mount type=bind,src="$PWD/backups",dst=/backup \
  -e BACKUP_NAME="$backup_name" \
  ubuntu:22.04 sh -c 'tar -czf "/backup/$BACKUP_NAME" -C /data .'
ls -lh "backups/$backup_name"
tar -tzf "backups/$backup_name"
sha256sum "backups/$backup_name" > "backups/$backup_name.sha256"
sha256sum -c "backups/$backup_name.sha256"
docker start volume-test2
```

확인 기준은 압축 파일이 생성되고, 목록에 `./message.txt`가 포함되며, 체크섬 검증이 `OK`를 반환하는 것이다. 백업용 컨테이너는 원본 볼륨을 읽기 전용으로 연결한다. 체크섬은 파일 손상 확인용이며, 데이터 복구 가능 여부는 다음 단계에서 별도로 검증한다.

**2. 새 볼륨으로 복구 및 내용 검증**

원본 `site-data`를 덮어쓰지 않고 새 볼륨으로 복구한다. 아래 예시는 같은 터미널에서 앞 단계의 `backup_name`을 사용한다. 터미널을 다시 열었다면 복구할 파일 이름으로 변수를 지정한다.

```bash
# 새 이름을 사용해 빈 복구 대상 볼륨 생성
restore_volume="site-data-restore-$(date +%Y%m%d-%H%M%S)"
docker volume create "$restore_volume"
sha256sum -c "backups/$backup_name.sha256"
# 위 검증이 OK일 때 복구 진행
docker run --rm \
  --mount type=volume,src="$restore_volume",dst=/data \
  --mount type=bind,src="$PWD/backups",dst=/backup,readonly \
  -e BACKUP_NAME="$backup_name" \
  ubuntu:22.04 sh -c 'tar -xzf "/backup/$BACKUP_NAME" -C /data'
docker run --rm \
  --mount type=volume,src="$restore_volume",dst=/data,readonly \
  ubuntu:22.04 cat /data/message.txt
```

실습 데이터가 정상적으로 복구되었을 때의 예상 출력:

```text
This data will survive container deletion!
```

내용을 확인한 후 서비스 컨테이너를 새 복구 볼륨에 연결해 사용한다. 복구 검증이 끝날 때까지 원본 볼륨과 백업 파일은 보관한다. `docker volume rm` 또는 `docker volume prune` 실행 전에는 삭제 대상을 확인한다.

**3. 호스트 백업 보관**

`backups/`는 호스트 디스크에 있으므로 Docker 볼륨이 삭제되어도 남는다. 다만 호스트 디스크 고장에는 대비할 수 없으므로 압축 파일과 체크섬 파일을 외장 저장장치나 별도 백업 저장소에 함께 복사한다. 날짜별 백업을 여러 개 유지하고, 중요한 변경 전과 정기 일정에 맞춰 백업한다. 개인정보·비밀정보가 포함된 백업은 공개 GitHub 저장소에 업로드하지 않는다.

## 10. Git 설정과 GitHub 연결

PowerShell에서 확인한 명령:

```powershell
git --version
git config --list
```

출력 발췌(개인 사용자 정보는 마스킹):

```text
git version 2.56.0.windows.4
init.defaultbranch=master
remote.origin.url=https://github.com/jeongb959-ai/codyssey_E1-1
branch.main.remote=origin
branch.main.merge=refs/heads/main
user.name=<마스킹>
user.email=<마스킹>
```

![Git 설정과 원격 저장소](image/git_config_list.png)

[제출 저장소](https://github.com/jeongb959-ai/codyssey_E1-1)

Git은 로컬에서 변경 이력과 브랜치를 관리하는 도구이고, GitHub는 원격 저장소를 공유하고 협업하는 플랫폼이다. 캡처에서는 사용자 정보, `origin`, `main`의 원격 추적 설정을 확인했다.

새 저장소의 기본 브랜치는 `master`로 설정되어 있으며, 이 저장소는 원격 `main`을 추적한다. 두 설정은 적용 대상이 다르므로 함께 존재할 수 있다. 사용자 정보는 이미지에서 마스킹했다.

### GitHub 커밋 이력

GitHub의 `main` 브랜치 커밋 목록에서 저장소 초기 생성과 실습 결과물 업로드 이력을 확인했다.

| 날짜 | 커밋 | 내용 |
| --- | --- | --- |
| 2026-11-03 | `301b393` | `Initial commit` — 저장소 초기 생성 |
| 2026-11-03 | `48f6f97` | `증명 스크린샷 및 작업물 업로드` — 실습 작업물과 증거 자료 업로드 |

![GitHub main 브랜치 커밋 목록](image/커밋목록.png)

Git 설정 화면의 원격 저장소 정보와 GitHub 커밋 목록을 통해 저장소 연결 설정 및 원격에 기록된 작업 이력을 확인할 수 있다.

## 11. 트러블슈팅

### 11.1 `echoo` 오타로 파일 내용 추가 실패 — 해결

- **문제:** 두 번째 줄을 추가할 때 `Command 'echoo' not found`가 출력되었다.
- **원인 가설:** `echo` 명령어를 `echoo`로 잘못 입력했다.
- **확인:** 입력값이 `echoo`였고 출력에 `Command 'echoo' not found`와 `echo` 추천 메시지가 표시되었다.
- **해결:** `echo "i am learning docker and git." >> terminal/practice.txt`로 다시 실행했다. `cat` 결과에 두 줄이 모두 표시되었다.

### 11.2 현재 브랜치와 기본 브랜치 설정 차이 — 설정 확인

- **문제:** 현재 저장소는 `main`을 추적하지만 `init.defaultbranch`는 `master`다.
- **원인 가설:** 기존 Git 전역 설정이 남아 있고, 복제한 저장소는 원격의 `main`을 사용한다.
- **확인:** [git config 캡처](image/git_config_list.png)에 `init.defaultbranch=master`와 `branch.main.merge=refs/heads/main`이 함께 있다.
- **해결 방법:** `git config --global init.defaultBranch main`을 실행하고 `git config --get init.defaultBranch`로 확인한다. 현재 저장소의 main 추적은 정상이며 기본 브랜치 이름 변경은 선택 사항이다.

### 11.3 로컬에 없는 이미지 다운로드 메시지 — 정상 동작 확인

- **상황:** Ubuntu 및 바인드 마운트 컨테이너의 첫 실행에서 `Unable to find image ... locally`가 출력되었다.
- **원인 가설:** 해당 태그의 이미지가 로컬에 없다.
- **확인:** 이어서 레이어 다운로드와 `Status: Downloaded newer image`가 출력되었다.
- **결과:** 다운로드 후 Ubuntu 쉘 실행 및 NGINX의 `HTTP/1.1 200 OK`를 확인했다. 별도 복구가 필요한 오류가 아니었다.

### 11.4 포트 충돌 진단 및 포트 해제 절차

**문제:** 웹 서버 실행 시 `port is already allocated` 또는 `address already in use`가 발생하면 호스트 포트를 다른 컨테이너나 프로세스가 사용 중인지 확인한다. 아래는 기존 `dev-web`의 8080 포트를 기준으로 한 진단·해제 절차다. 이 절의 출력은 동작을 설명하기 위한 예상 출력 예시이며, 실제 실행 로그는 아니다. PID와 프로세스 이름은 환경에 따라 달라진다.

**1. 포트 사용 주체 확인**

WSL에서 리스닝 소켓과 프로세스 정보를 확인한다. 프로세스 정보가 표시되지 않으면 권한이 허용되는 환경에서만 `sudo`를 사용한다.

```bash
ss -ltnp 'sport = :8080'
# lsof가 설치되어 있다면 교차 확인
lsof -nP -iTCP:8080 -sTCP:LISTEN
# Docker 컨테이너의 호스트 포트 매핑 확인
docker ps --filter publish=8080 --format 'table {{.Names}}\t{{.Ports}}\t{{.Status}}'
```

Linux 호스트에서 Docker 포트를 프록시 프로세스가 점유하는 경우의 예상 출력 예시:

```text
$ ss -ltnp 'sport = :8080'
State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
LISTEN 0      4096   0.0.0.0:8080      0.0.0.0:*       users:(("docker-proxy",pid=12345,fd=4))

$ docker ps --filter publish=8080 --format 'table {{.Names}}\t{{.Ports}}\t{{.Status}}'
NAMES     PORTS                   STATUS
dev-web   0.0.0.0:8080->80/tcp   Up 5 minutes
```

이 예에서는 호스트 8080 포트가 이미 `dev-web`에 연결되어 있으므로 다른 컨테이너가 같은 호스트 주소의 8080 포트를 사용할 수 없다. Docker의 네트워크 구현에 따라 `docker-proxy`가 표시되지 않을 수도 있으므로 컨테이너 포트 매핑도 함께 확인한다.

`ss`의 `LISTEN`, 로컬 주소의 `:8080`, `users`의 프로세스/PID를 확인한다. Docker Desktop이 Windows 쪽에 포트를 게시하면 WSL의 `ss`에는 나오지 않을 수 있다. 이 경우 Windows PowerShell에서 다음을 확인한다.

```powershell
netstat -ano -p tcp | Select-String ':8080\s'
# 출력의 로컬 주소가 8080이고 LISTENING인 행의 PID를 조회
$ownerPid = 12345 # 실제 LISTENING 행에서 확인한 PID로 변경
Get-Process -Id $ownerPid
```

Windows의 예상 출력 예시:

```text
TCP    0.0.0.0:8080    0.0.0.0:0    LISTENING    12345
```

Docker Desktop 관련 프로세스가 표시되면 해당 프로세스를 종료하지 않고, `docker ps`의 포트 매핑으로 점유 컨테이너를 찾는다. 외부 연결의 원격 포트가 8080인 행을 로컬 포트 점유로 혼동하지 않는다.

**2. 점유 컨테이너 중지 → 포트 해제 확인**

다음 명령은 점유 컨테이너가 실습용 `dev-web`임을 확인한 경우에 실행한다. `stop`은 컨테이너를 삭제하지 않으며, 데이터 보존을 위해 볼륨 삭제 명령은 사용하지 않는다.

```bash
docker stop dev-web
docker ps --filter publish=8080 --format 'table {{.Names}}\t{{.Ports}}'
ss -ltnp 'sport = :8080'
```

정상적으로 해제되었을 때의 예상 출력 예시:

```text
$ docker stop dev-web
dev-web
$ docker ps --filter publish=8080 --format 'table {{.Names}}\t{{.Ports}}'
NAMES     PORTS
$ ss -ltnp 'sport = :8080'
State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
```

목록에 컨테이너 행이 없고, `ss`에는 헤더만 표시되는 상태를 확인한다. Windows의 `netstat` 필터는 해당 행이 없어 출력이 비어 있어야 한다.

`docker ps`에 실행 중인 점유 컨테이너가 없고 `ss`에 해당 리스너가 없는지 확인한다. Windows에서 포트를 점유했던 경우에는 PowerShell의 `netstat`도 다시 실행하여 `LISTENING` 행이 사라졌는지 확인한다. 컨테이너가 아닌 일반 프로세스라면 PID와 프로그램의 용도를 확인한 뒤 해당 앱이나 서비스의 정상 종료 기능으로 중지한다.

**3. 원하는 컨테이너 실행 → 접속 확인**

```bash
docker start dev-web
docker ps --filter name=dev-web
curl -I http://localhost:8080
```

재시작 후 예상 출력 예시:

```text
$ docker start dev-web
dev-web
$ curl -I http://localhost:8080
HTTP/1.1 200 OK
Server: nginx/...
Content-Type: text/html
```

확인 기준은 `dev-web`의 실행 상태와 `8080->80/tcp` 매핑, HTTP 성공 응답이다. 기존 서비스를 계속 유지해야 한다면 점유 서비스를 종료하는 대신 사용하지 않는 호스트 포트로 새 컨테이너를 실행한다.

```bash
# 8082가 비어 있는지 먼저 확인
ss -ltnp 'sport = :8082'
docker ps --filter publish=8082
docker run -d --name dev-web-alt -p 8082:80 developer-workstation-web:1.0
curl -I http://localhost:8082
```

Windows 측 포트 게시 환경에서는 8082도 `netstat`로 함께 확인한다. 해제 순서는 **점유 주체 확인 → 해당 컨테이너/앱 정상 중지 → 리스너 소멸 확인 → 서비스 실행 → HTTP 응답 확인**이다.
