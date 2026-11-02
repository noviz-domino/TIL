---
tags: [docker, compose, postgresql, pgvector, fastapi]
til: v2 2026-10-30
---

# 2026-10-30_Docker와_Compose_실습(WSL2,_볼륨,_pgvector,_FastAPI,_바인드마운트)
> 작성일: 2026-10-30

## 🔗 관련 글

- [FastAPI 시작과 Swagger UI](2026-10-23%20(2)FastAPI_시작(uv,_엔드포인트,_Swagger_UI,_자동_문서화).md) — 로컬 API 실행과 문서 테스트를 컨테이너 실행으로 확장한다.
- [SQLAlchemy와 PostgreSQL 게시판 실습](2026-10-27_SQLAlchemy와_PostgreSQL_게시판_실습(Engine,_Session,_CRUD,_Pydantic).md) — 처음에는 게시판 DB 연결을 확인했고, 이후 실제 수업의 메모·검색 backend로 전환했다.
- [FastAPI 비동기 DB와 Alembic](2026-10-29%20(1)FastAPI_비동기_DB와_Alembic_마이그레이션(lifespan,_관계_로딩).md) — 이미지 교체와 스키마 변경의 차이, lifespan의 DB 준비와 연결한다.
- [동기와 비동기의 실행 구조](2026-10-29%20(3)동기와_비동기의_실행구조(이벤트루프,_스레드풀,_스케줄링,_CPU,_코루틴).md) — Windows Psycopg 비동기 연결에서 이벤트 루프 설정이 필요한 이유를 복습한다.
- [RAG 임베딩·검색과 기본 구조](2026-10-16%20(1)RAG_Agent_%EC%8B%A4%EC%8A%B52_%EC%9E%84%EB%B2%A0%EB%94%A9_%EA%B2%80%EC%83%89_%EA%B8%B0%EB%B3%B8_RAG(Chroma,_build_index,_search,_LCEL).md) — 이번 pgvector 실습은 검색 자료의 저장·검색 구현을 PostgreSQL로 바꾸는 부분에 해당한다.

## ❓ 아직 모르겠는 것

다음은 실제 질문·반박·이해 확인에 근거한 목록이다. 대화에서 풀린 것도 복습 때 다시 확인한다. 조교 설명이 불명확해 생긴 혼란을 사용자의 실수로만 취급하지 않는다.

- “도커가 뭐지?”, “환경을 구분하는 개발자 도구야?” — 환경 격리와 서비스 배포는 어떻게 연결되는가? (주제 1·12)
- “OS 자체를 배포해?”, “서버가 커널만 제공해?”, “실제 실행은 서버구나.” — 커널·사용자 공간·자원·실행 위치를 구분할 수 있는가? (주제 1·2)
- “엔진과 데스크톱 둘 다 설치돼?”, “엔진 아래 데스크톱 아니었어?”, “엔진이 백엔드?” — 포함 관계와 요청 전달 관계가 왜 다른가? (주제 3)
- “Hub는 GitHub처럼 아무나 올려?”, “해킹당할 수도 있네”, “완전히 격리돼?” — 출처·권한·격리의 한계는 무엇인가? (주제 4)
- “WSL이면 다 Ubuntu야?”, “Ubuntu와 docker-desktop은 왜 둘?”, “설치된 Ubuntu에서 돌아?” — 세 층위가 어떻게 다른가? (주제 5)
- “전용 Linux가 무슨 Linux인지 모르냐?”, “OS 지정 안 하면 Docker Linux?” — 확인한 배포판 이름과 이미지 선택을 구분하는가? (주제 5)
- “설정 코드를 수정하면 돼?”, “VS Code에 파일 만들고 다음엔?”, “Desktop에서 추가 못 해?” — 설정 작성·CLI 실행·GUI 조작의 차이는 무엇인가? (주제 6·7)
- “GET /health가 없는데?”, “어디 연결됐는지 모르잖아.” — 실행 앱과 실제 DB 접속 대상을 어떻게 검증하는가? (주제 8)
- “볼륨 범위는?”, “UI에서 삭제하면?”, “postgres_data:는 왜 비어 있어?” — 연결 경로·선언 옵션·실제 데이터의 차이는 무엇인가? (주제 9)
- “up/down이 뭐야?”, “이미지가 있으니 다시 만드는 게 빠른가?” — 재시작과 재생성에서 무엇이 남는가? (주제 10)
- “VM 복제처럼 n배 용량?”, “수천·수만 명 서비스에 부적절?” — 이미지 공유와 실행 자원은 어떻게 다른가? (주제 11·13)
- “DB는 같아야 하잖아. Hadoop도 아닌데?” — 앱 A·B·C와 공통 DB는 어떤 구조인가? (주제 13)
- “이미 환경이 있는데 왜 도커?”, “자원만 차지하지?”, “운영 중인 회사에는 의미 없지?” — 도입의 비용과 이점은 무엇인가? (주제 12)
- “사용자는 차이 없지?”, “아까 A·B·C는?” — 사용자 경험과 서버 운영 구성을 구분하는가? (주제 12·13)
- “라이브러리 바뀌면 안 되나? 데이터 구조도 따라가나?” — 실행 환경 교체와 마이그레이션은 어떻게 다른가? (주제 14)
- “Mac과 Windows가 안 맞아?” — 이번 호환 설정의 대상과 실제 검증 범위는 무엇인가? (주제 17)
- “condition에는 무슨 값?”, “즉 0?”, “위에서부터 실행돼?” — 종료 코드·건강 상태·YAML 조건의 관계는 무엇인가? (주제 16)
- “바인드 마운트란?”, “그냥 재시작 아냐?”, “이미지가 빈 껍데기?” — 파일 연결과 reload를 왜 구분하는가? (주제 18)
- “운영 A·개발 B가 같은 이미지면 껐다 켜면 예전 코드?” — 마운트 유지·제거 시 실행 파일이 어떻게 달라지는가? (주제 18)
- “선행 정보가 없었어? 왜 이해가 안 되지?” — 파일·프로세스·재시작 차이부터 설명해야 했다. (주제 18·20)
- Docker 전용 배포판의 내부 기반 계보는 무엇인가? — 이름과 커널은 확인했으나 Ubuntu·Alpine 등 기반 계보까지는 확인하지 않았다. (미해결, 주제 5)

## 주제 1: Docker가 해결하는 문제와 실행 위치

Docker는 앱과 필요한 파일·설정을 이미지로 준비하고 그 이미지에서 컨테이너를 생성·실행·관리하는 플랫폼이다. 코드만 전달하면 실행자가 Python·패키지·시스템 라이브러리를 직접 설치해야 한다. 이미지에는 실행 파일·라이브러리·코드·시작 명령을 함께 준비할 수 있다.

| 환경 차이 | 발생 가능한 문제 |
|---|---|
| Python 버전 | 지원 문법·패키지 차이 |
| 라이브러리 버전 | 함수 사용법·결과 차이 |
| 여러 프로젝트 공존 | 요구 버전·프로그램 충돌 |
| 다른 컴퓨터에 환경 준비 | 설치·설정 누락 |

같은 이미지를 사용하면 프로젝트별 환경 준비를 반복하는 일을 줄인다. PostgreSQL을 이미지로 실행하면 실습용 DB 서버를 호스트에 직접 설치하는 과정도 줄일 수 있다.

> ➕ **더 알아두기**
> 도커는 원격 서버만 제공하는 서비스가 아니다. 컨테이너를 실행한 컴퓨터의 CPU·메모리로 작동한다. 노트북에서 실행하면 노트북에서, 클라우드 서버에서 실행하면 서버에서 돌아간다. 개발에서는 환경 분리, 배포에서는 준비한 이미지의 전달·재실행에 활용한다. CPU 구조·OS 호환 조건까지 무조건 같아지는 것은 아니다.

> ♻️ 처음엔 환경만 제공하고 실행은 항상 서버에서 한다고 알았으나, 실제 실행 장소는 컨테이너를 실행한 컴퓨터라는 것이 맞다.

## 주제 2: 커널·OS·VM·컨테이너와 .venv

컨테이너는 프로그램 실행 환경을 격리하는 단위다. 서로 다른 컨테이너에서 Python 3.11·3.12나 서로 다른 패키지를 사용할 수 있다.

| 구분 | Python 가상환경 .venv | Docker 컨테이너 |
|---|---|---|
| 목적 | 프로젝트별 Python 패키지 분리 | 앱 실행 환경 구성과 격리 |
| 파일·프로세스·네트워크 | 호스트 환경 사용 | 격리된 환경 구성 |
| PostgreSQL 같은 별도 서버 | 가상환경만으로 설치·격리하지 않음 | 해당 이미지로 실행 가능 |

가상환경과 컨테이너는 함께 쓸 수 있다. 이번 Dockerfile도 컨테이너 안에서 uv로 .venv를 만든다.

> ➕ **더 알아두기**
> 커널(Kernel)은 CPU·메모리·파일·네트워크 같은 자원을 관리하는 OS의 핵심이다. 명령어·시스템 라이브러리·파일 구조 같은 사용자 공간과 구분한다. Linux 컨테이너 이미지는 사용자 공간 파일들을 포함할 수 있지만, 컨테이너마다 커널을 하나씩 실행하지는 않는다. VM은 보통 별도의 게스트 OS·커널을 가지며, 컨테이너는 실행 기반의 커널을 공유한다.
>
> 서버는 커널만 제공하는 것이 아니라 실제 CPU·메모리·디스크·네트워크도 제공한다. Windows에서 Linux 컨테이너가 Windows 커널로 직접 실행되는 것도 아니다. 현재는 WSL 2가 Linux 커널을 제공한다.

> ♻️ 처음엔 OS 전체를 가진 작은 VM처럼 생각했으나, 컨테이너별 사용자 공간 격리와 커널 공유를 구분하는 것이 맞다.

> 📝 **기사 시험 포인트** (정보처리기사·정보보안기사)
> 가상화(Virtualization), 커널(Kernel), 프로세스(Process), 자원 격리(Isolation)를 구분한다. 격리는 상호 영향을 제한하는 기능이며 자원 경합·모든 공격의 부재를 보장하는 말은 아니다. 구체적인 Docker 명령의 출제 빈도는 단정하지 않는다.

## 주제 3: Engine·Desktop·CLI의 관계

수업의 설치 단계는 Windows에서는 CPU·시스템 요구 사항에 맞는 Docker Desktop 공식 설치 안내를 따르고, macOS에서는 Homebrew가 준비되어 있으면 다음 명령을 사용하는 흐름이다.

```bash
brew install --cask docker-desktop
```

설치 후 Desktop을 실행하고 Engine 시작이 끝날 때까지 기다린다. 이 컴퓨터에서는 이미 설치된 상태를 확인했으므로 재설치하지 않았다.


| 구성 요소 | 역할 |
|---|---|
| Engine | 이미지·컨테이너·네트워크·볼륨 관리와 실행 처리 |
| CLI | 터미널의 docker 명령으로 Engine에 요청하고 결과 표시 |
| Desktop | 로컬 사용을 위한 실행 기반·Engine·CLI·Compose·관리 화면 제공 |

CLI가 설치되어 있어도 Engine에 연결할 수 없으면 컨테이너를 실행할 수 없다. Windows에서는 보통 Desktop을 실행해 Engine을 준비한다.

```text
포함·제공 관계: Desktop → 관리 화면 + CLI/Compose + Engine 실행 환경
요청 전달 관계: 관리 화면 또는 docker 명령 → Engine → 컨테이너 작업
```

> ➕ **더 알아두기**
> Engine이 백엔드, Desktop 화면이 프론트엔드라는 비유는 요청과 처리 역할을 이해하는 데 유용하다. 다만 Desktop 전체는 화면만이 아니라 Engine과 실행 기반도 제공한다. Git Bash·CMD는 명령을 입력하는 터미널이고 Docker CLI는 그 안에서 쓰는 도구다. 앞선 설명에서 포함 관계와 화살표 흐름을 혼용해 혼란이 생겼다.

> ♻️ 처음엔 환경 각각이 Engine이라고 알았으나, 각각의 실행 공간은 컨테이너이고 Engine이 여러 컨테이너를 관리하는 것이 맞다.

## 주제 4: Docker Hub·이미지 출처·보안

이미지 레지스트리(Image Registry)는 이미지를 저장·배포하는 서버다. Docker Hub는 그 서비스 중 하나로 Python·PostgreSQL 이미지 등을 공유한다. 태그(Tag)는 이미지에 붙인 이름표다. postgres:17의 17은 버전 계열 선택에 쓰지만, 같은 태그가 나중에 다른 이미지 내용을 가리킬 수 있다.

> ➕ **더 알아두기**
> GitHub에 코드를 올리듯 Hub에는 계정·권한·서비스 정책에 맞춰 이미지를 올릴 수 있다. Desktop은 로컬 앱, Hub는 온라인 저장소다. 다른 사람의 이미지는 실행 가능한 프로그램을 포함하므로 출처가 중요하다. 공식 이미지·신뢰할 수 있는 제작자를 확인하고 출처 모르는 명령과 과도한 권한을 그대로 사용하지 않는다. 공식 표시도 모든 취약점 부재의 보증은 아니다.
>
> 컨테이너는 완벽한 방어막이 아니다. 커널과 자원을 공유하고, 호스트 폴더를 쓰기 가능으로 연결하면 그 파일을 수정·삭제할 수 있다. 취약점으로 경계를 벗어나는 컨테이너 탈출(Container Escape)도 가능하다. “한 건물의 분리된 방” 비유를 절대 안전의 의미로 해석하지 않는다.

> 📝 **기사 시험 포인트** (정보보안기사)
> 최소 권한 원칙(Principle of Least Privilege), 접근통제(Access Control), 취약점 관리(Vulnerability Management)를 연결한다. 프로그램의 출처와 실행 권한은 별도로 검토한다.

## 주제 5: 내 Windows·WSL 2·Ubuntu·docker-desktop

> ➕ **더 알아두기 — 실제 확인한 환경**
> 당시 Docker CLI/Engine은 29.7.2, 선택 컨텍스트는 desktop-linux였다. WSL에는 Ubuntu와 docker-desktop이 각각 버전 2로 등록되어 있었다. Docker는 Linux 컨테이너 모드였고 커널은 6.18.33.2-microsoft-standard-WSL2였다. 버전·상태는 이후 바뀔 수 있다.

```bash
wsl --list --verbose
docker version
docker context ls
docker info
docker ps -a
```

> ➕ **더 알아두기 — 실행 층위를 구분한다**
> WSL 2는 Windows에서 Linux 커널을 사용하는 실행 기반이다. Ubuntu와 docker-desktop은 서로 별도의 WSL 배포판이다. 현재 Docker Engine은 내 Ubuntu가 아니라 전용 docker-desktop에서 돌아간다. 개인용 Ubuntu를 켜야 Docker가 작동하는 구조가 아니며 별도 Ubuntu 설치가 필수도 아니다. Ubuntu가 설치된 정확한 경위는 확인하지 않았다.
>
> 컨테이너 안의 파일·라이브러리 환경은 이미지가 정한다. FROM ubuntu:24.04는 Ubuntu 기반, FROM python:3.12는 Python 이미지 제작자가 정한 환경을 사용한다. OS 이름을 직접 쓰지 않았다고 Docker 전용 배포판의 파일이 자동으로 컨테이너에 들어가는 것은 아니다.

```text
Windows
└─ WSL 2 기반
   ├─ Ubuntu: 별도로 쓰는 Linux 배포판
   └─ docker-desktop: Docker 전용 Linux 배포판
      └─ Engine이 관리하는 컨테이너
         └─ 선택한 이미지의 프로그램·라이브러리·파일
```

> ➕ **더 알아두기 — 확인한 것과 모르는 것**
> docker-desktop의 /etc/os-release를 읽은 결과 PRETTY_NAME은 Docker Desktop이었다. 이 값만으로 내부 기반 계보가 Ubuntu·Alpine 중 무엇인지 단정할 수 없다. WSL 2 커널은 별도로 확인했다. 이 부분은 마지막까지 구체적인 기반 배포판을 확정하지 않았다.

> ♻️ 처음엔 WSL이면 모든 컨테이너가 내 Ubuntu에서 돈다고 알았으나, Engine의 전용 배포판과 컨테이너 이미지 환경은 별개라는 것이 맞다.

## 주제 6: Dockerfile·Compose·프로젝트·서비스와 YAML

Dockerfile은 이미지에 넣을 프로그램·파일과 준비 방법을 정의한다. 빌드(Build)는 그 파일을 읽어 이미지를 만드는 과정이다. Compose는 이미지·환경변수·포트·네트워크·저장 공간 등 컨테이너 실행 구성을 정의하고 관리한다.

| 파일 | 답하는 질문 |
|---|---|
| Dockerfile | 어떤 실행 환경과 코드를 담은 이미지를 만들까? |
| compose.yaml | 어떤 컨테이너를 어떤 설정으로 실행·연결할까? |
| .dockerignore | 이미지 빌드에 보내지 않을 파일은 무엇인가? |

Compose가 여러 서비스를 관리한다고 하나의 이미지로 합쳐지지 않는다. db와 backend는 별도 컨테이너다. 서비스 하나만으로도 Compose를 사용할 수 있다.

```yaml
name: docker-practice

services:
  db:
    image: postgres:17
```

name은 프로젝트 이름으로 컨테이너·네트워크·볼륨을 구분한다. services는 서비스 설정 모음, db는 직접 정한 서비스 이름, image는 사용할 이미지다. YAML은 들여쓰기로 계층을 정하며 탭 대신 공백 2칸을 사용한다.

> ➕ **더 알아두기**
> Desktop의 Images → Run → Optional settings에서도 포트·환경변수·마운트를 넣어 실행할 수 있다. 수업은 설정을 코드와 함께 보관하고 재현하기 위해 Compose를 사용한다. 생성 후 Desktop에서 시작·중지·로그 확인도 가능하다.
>
> YAML은 Python처럼 위에서부터 실행되는 프로그램이 아니다. Compose가 파일을 읽고 의존 관계에 따라 순서를 정한다. db보다 backend를 먼저 적었다고 backend가 먼저 준비되는 것은 아니다.

## 주제 7: PostgreSQL 초기 설정과 포트

볼륨 추가 전 DB 설정은 다음과 같다.

```yaml
name: docker-practice

services:
  db:
    image: postgres:17
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: "<DB_PASSWORD>"
      POSTGRES_DB: docker_prac
    ports:
      - "127.0.0.1:5433:5432"
```

| 설정 | 의미 |
|---|---|
| POSTGRES_USER | 초기 DB 사용자 |
| POSTGRES_PASSWORD | 그 사용자의 초기 비밀번호 |
| POSTGRES_DB | 처음 만들 DB |
| ports | 호스트 주소·포트를 컨테이너 포트에 연결 |

이 값들은 빈 데이터 디렉터리를 처음 초기화할 때 사용한다. 기존 볼륨에 계정이 있는데 YAML의 비밀번호만 바꿔도 계정 비밀번호가 바뀌는 것은 아니다. <DB_PASSWORD>는 로컬 실습용 값이다.

```text
127.0.0.1 : 5433 : 5432
호스트 주소   호스트 포트   컨테이너 포트
```

호스트는 여기서는 내 Windows다. 기존 PostgreSQL이 호스트 5432를 사용하므로 실습 DB는 5433으로 공개했다. DB 컨테이너 내부 포트는 5432다. VS Code DB 연결 도구에서도 127.0.0.1, 5433, postgres, docker_prac으로 접속한다.

```bash
docker compose up -d
docker compose ps
docker compose logs db
```

up은 필요하면 이미지를 받아 컨테이너를 생성·실행한다. -d는 백그라운드 실행, ps는 상태, logs db는 DB 로그다. database system is ready to accept connections를 준비 완료 신호로 확인한다. Desktop에서 docker-practice를 펼쳐 db 로그를 볼 수도 있다.

> ➕ **더 알아두기 — VS Code에서 처음 시작하는 흐름**
> 프로젝트 폴더를 열고 최상위에 compose.yaml을 만들어 저장한다. Git Bash의 위치를 pwd와 ls compose.yaml로 확인한 뒤 up을 실행한다. 파일 저장만으로 서버가 실행되지는 않는다. 처음 fastapi_agent로 잘못 짚었고 사용자가 board-practice로 정정했으며, 이후 실제 수업 backend 위치를 알려 주어 전환했다. 최종 실습 경로는 주제 17에 기록한다.

> 📝 **기사 시험 포인트** (정보처리기사·정보보안기사)
> IP 주소(IP Address)와 포트 번호(Port Number)를 구분한다. 루프백 주소(Loopback Address) 127.0.0.1은 자기 실행 환경을 가리킨다. 컨테이너와 호스트의 루프백은 각각 다른 환경이다.

## 주제 8: 앱 실행과 DB 연결을 검증하는 방법

수업 backend는 시작 시 lifespan에서 notes 테이블을 준비하고 PostgreSQL로 저장·조회한다.

| API | 기능 |
|---|---|
| GET /health | 앱·DB 연결 확인 |
| POST /notes | 메모 저장 |
| GET /notes | 최근 메모 최대 20개 조회 |
| POST /search | 임베딩 텍스트 검색 |

```dotenv
DATABASE_URL=postgresql://postgres:<DB_PASSWORD>@127.0.0.1:5433/docker_prac
GEMINI_API_KEY=발급받은_API_키
```

```bash
uv sync
uv run fastapi dev main.py
```

http://127.0.0.1:8000/docs에서 API를 펼쳐 Try it out → Execute로 요청한다. health 후 메모를 저장하고 GET /notes에서 같은 id와 text를 확인한다.

```json
{
  "text": "Compose로 PostgreSQL 연결 성공"
}
```

> ➕ **더 알아두기 — 실제 앱이 수업과 달랐던 문제**
> 사용자가 /health가 없다고 해서 문서를 열어 보니 board-practice의 게시글·댓글·태그 앱이었다. /health는 FastAPI가 자동 생성하는 기능이 아니라 직접 구현하는 경로다. GET /posts는 200과 게시글 1개를 반환했지만, 그 사실만으로 도커 DB를 사용한다고 확정할 수 없었다.
>
> .env가 localhost:5432/board_practice였고 도커 DB는 5433/docker_prac이었다. 계정·비밀번호가 같아도 접속 대상은 다르다. 요청에 따라 주소를 바꾸고 실제 쿼리를 보내는 /health를 추가했다. 실패 시 HTTP 503도 검증했다. 연결 대상이 안 보인다는 후속 질문으로 host·port와 실제 current_database(), current_user 결과를 응답에 추가했다. 비밀번호는 표시하지 않는다.
>
> .env를 바꿔도 실행 중인 프로세스는 이전 값을 계속 쓸 수 있어 서버를 재시작했다. localhost 접속 지연을 피하려고 로컬 접속 주소를 127.0.0.1로 명시한 뒤 정상 동작을 확인했다. 최종 컨테이너 health는 host=db, port=5432, name=docker_prac, user=postgres였다. health 성공은 모든 업무 테이블·기능이 정상이라는 보증과 다르다.

> ♻️ 처음엔 /health가 기본으로 생기는 줄 알았으나, 실행 중인 앱에 해당 경로를 구현해야 한다는 것이 맞다.

## 주제 9: 볼륨의 범위·삭제와 빈 postgres_data 선언

이미지는 실행할 프로그램을 준비하고 볼륨(Volume)은 실행 중 생기는 데이터를 컨테이너와 별도로 보관한다. 컨테이너 자체의 쓰기 공간은 stop/start에는 남지만 삭제하면 변경분도 사라진다. DB를 교체해도 보존할 데이터는 볼륨에 둔다.

```yaml
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

| 부분 | 의미 |
|---|---|
| db 아래 volumes | 컨테이너에 연결할 저장 공간 |
| postgres_data | 사용할 볼륨의 이름 |
| /var/lib/postgresql/data | PostgreSQL 17 내부 데이터 경로 |
| 최상위 volumes | 사용할 이름 있는 볼륨 선언 |

범위는 어느 컨테이너 경로를 연결할지다. 개별 테이블만 고르는 설정이나 용량 상한 설정이 아니다. 해당 PostgreSQL 데이터 폴더의 테이블·데이터 등을 보관한다. 실제 생성 이름은 docker-practice_postgres_data였다.

> ➕ **더 알아두기 — 왜 postgres_data: 아래가 비었나?**
> 추가 옵션 없이 기본 드라이버·설정으로 만들라는 뜻이다. 이 용도에서는 postgres_data: {}와 같은 의미다. 설정 파일에는 이름·옵션을 적고 실제 데이터는 Docker가 관리하는 디스크에 저장한다. 빈 선언과 빈 데이터는 다르다.
>
> 볼륨을 삭제하면 안의 데이터도 사라진다. Desktop의 긴 무작위 이름도 실제 볼륨일 수 있다. 당시 긴 이름의 볼륨은 PostgreSQL 데이터 경로에 연결된 익명 볼륨임을 inspect로 확인했다. 이름만 보고 불필요하다고 판단하지 않는다. Docker는 일반적으로 정지된 컨테이너를 포함해 연결된 볼륨 삭제를 막는다.
>
> 볼륨은 백업(Backup)이 아니다. DB에서 삭제한 데이터는 볼륨에도 삭제 상태로 반영된다. 볼륨 삭제·손상·디스크 장애에 대비하려면 별도 백업이 필요하다.

수업은 볼륨 설정 전에 메모 저장 → down/up → 이전 메모가 안 보이는지 비교한 뒤 이름 있는 볼륨을 추가하는 흐름이다. 새 빈 볼륨을 연결하면 새 DB로 시작한다.

> ➕ **더 알아두기 — 익명 볼륨에 대한 정확한 구분**
> PostgreSQL 이미지는 데이터 경로에 익명 볼륨을 자동 생성할 수 있다. down 후 이 볼륨이 디스크에 남더라도 새 컨테이너가 같은 익명 볼륨을 자동 연결한다고 기대할 수 없다. “메모가 조회되지 않는다”와 “볼륨 파일이 전부 삭제됐다”는 다른 말이다.
>
> 실제 수행 시 기존 DB에 관계 테이블은 없었다. 익명 볼륨은 삭제하지 않고 이름 있는 볼륨을 연결한 뒤 게시판 테이블을 만들었다. /posts에 제목 “볼륨 저장 실습”, 내용 “컨테이너를 다시 만들어도 남을 메모”를 저장해 id 1과 조회를 확인했다. 실제 backend로 옮긴 뒤 /notes에도 같은 실습 메모를 저장했다. 볼륨 적용 전 메모를 일부러 없애는 비교 절차를 실제 수행했다고 기록하지 않는다.

> 📝 **기사 시험 포인트** (정보처리기사·정보보안기사)
> 영속성(Persistence), 백업(Backup), 복구(Recovery)를 구분한다. 실행 단위를 교체해도 데이터가 남는 것과 손실에 대비한 복사본이 존재하는 것은 다르다.

## 주제 10: up·down·stop·start·재생성

| 명령 | 동작 |
|---|---|
| docker compose up -d | 설정에 맞게 생성·실행, 필요 시 변경된 컨테이너 재생성 |
| docker compose ps | 현재 프로젝트 상태 |
| docker compose logs db | DB 로그 |
| docker compose stop | 컨테이너를 남기고 중지 |
| docker compose start | 남아 있는 중지 컨테이너 시작 |
| docker compose restart db | 기존 DB 컨테이너 재시작 |
| docker compose down | 프로젝트 컨테이너와 기본 네트워크 제거 |

일반 down은 이미지와 이름 있는 볼륨을 삭제하지 않는다. down -v는 볼륨 삭제를 포함하므로 지금 실습 데이터 보존 시 붙이지 않는다. restart/start는 새 환경변수·마운트 설정을 적용하는 재생성과 다르다.

수업의 재생성 검증은 로컬 FastAPI 중지 → down → up → DB 준비 확인 → FastAPI 재실행 → GET /notes의 같은 id·text 확인이다. Desktop Volumes에서도 이름 있는 볼륨을 확인한다.

> ➕ **더 알아두기**
> 로컬 이미지가 있으면 매번 PostgreSQL을 새로 설치하지 않는다. 이미지에서 새 컨테이너를 만들고 기존 볼륨을 연결한다. 생성은 대체로 빠르지만 DB 준비 완료까지는 별도 시간이 필요하다. down/up은 이전 컨테이너를 지우고 새로 만드는 흐름이므로 개수가 계속 누적되지 않는다.
>
> 실제로는 pgvector 이미지 전환과 healthcheck 추가 과정에서 DB가 재생성되었다. 그 후 기존 메모·게시글을 읽어 보존을 확인했다. 컨테이너 실행과 데이터 보존 검증을 별도로 확인한 것이다.

## 주제 11: VM처럼 복제되나? 디스크·CPU·메모리

> ➕ **더 알아두기 — 실제 사용자 질문에서 확장**
> 같은 이미지에서 여러 실행 환경을 만든다는 점은 VM 복제와 비슷하다. 하지만 이미지 전체를 컨테이너마다 통째로 복사하는 것은 아니다. 읽기 전용 이미지 레이어(Layer)는 공유하고 각 컨테이너의 변경분은 별도 쓰기 레이어에 기록한다.
>
> 예를 들어 이미지 500MB, 컨테이너 세 개의 변경분이 각각 10MB라면 단순화한 합계는 약 530MB와 관리 공간이다. 무조건 500 × 3MB가 아니다. 실제 변경 시 파일 복사 등으로 사용량은 작업과 저장 방식에 따라 달라진다.
>
> 디스크 공유와 실행 비용은 다르다. 프로그램마다 CPU·메모리·DB 연결을 사용하고 볼륨·로그·파일 변경도 공간을 차지한다. 별도 볼륨을 여러 개 만들면 각각의 데이터 크기만큼 용량이 필요하다. 도커가 실행 자원을 무료로 만들어 주는 것은 아니다.

> ♻️ 처음엔 컨테이너를 n개 만들면 이미지 디스크도 무조건 n배가 되는 것으로 이해했으나, 이미지 레이어 공유와 개별 변경분·볼륨·실행 자원을 구분하는 것이 맞다.

## 주제 12: 이미 준비된 서버·회사·사용자에게 도커의 의미

> ➕ **더 알아두기 — 도커 도입의 타당성을 따진 대화**
> 한 클라우드 컴퓨터에 필요한 환경이 이미 있고 변경도 거의 없다면 도커 없이 서비스할 수 있다. 직접 설치·관리하는 방식도 합리적이며 도커는 필수가 아니다. 버전이 무조건 저절로 바뀌기 때문에 쓰는 것도 아니다.
>
> 도커의 비용도 있다. 이미지·볼륨 디스크, 관리 복잡성, Windows에서는 WSL 2 실행 기반의 자원이 필요하다. Linux 서버는 기존 Linux 커널을 공유하므로 Windows에서 Linux 실행 기반을 추가하는 경우와 구조가 다르다. 정확한 성능·비용은 실제 부하에서 측정해야 한다.
>
> 주된 이점은 반복 가능한 준비와 교체다. 새 서버 설치, 프로젝트별 버전 분리, 테스트와 운영에서 검증한 같은 이미지 사용, 여러 서버에 같은 앱 배포, 실행 환경 버전 관리, 문제 시 이전 앱 이미지로 되돌리는 작업을 다루기 쉽다.
>
> 이미 서비스하는 회사도 업데이트·증설·복구를 계속 수행한다. 현재 설치가 잘 되어 있다는 것이 향후 버전과 서버의 준비까지 끝났다는 뜻은 아니다. 하지만 기존 배포 방식이 충분히 잘 작동하고 변경이 적다면 전환 이점이 작을 수 있다. 회사라서 무조건 써야 하는 기술은 아니다.
>
> 우리 실습에서는 Windows에 pgvector를 직접 설치하지 않고 준비된 이미지를 쓰고, 기존 DB와 포트를 분리했으며, 나중에는 FastAPI까지 이미지로 만들었다. 도커가 앱의 성능·가용성을 자동으로 높여 주는 것은 아니다.
>
> 웹 사용자는 서버의 데이터·화면을 받는다. 같은 기능과 응답이라면 도커 사용 여부로 사용 방법이 바뀌지 않고 도커 설치도 필요 없다. 배포 중단·복구 속도 같은 운영 품질은 간접적으로 체감할 수 있다.

## 주제 13: 사용자 수·A/B/C 앱·공통 DB

> ➕ **더 알아두기**
> 사용자 한 명마다 컨테이너 하나를 만드는 것이 아니다. FastAPI 한 서버도 여러 요청을 처리한다. 처리량을 늘리려면 같은 앱을 여러 컨테이너·프로세스·컴퓨터에 실행하고 로드밸런서(Load Balancer, 요청 분산 장치)가 요청을 나눌 수 있다. 가입자 1만 명과 동시 요청 1만 개는 다르며 필요한 서버 수는 요청량·작업량·자원에 따라 달라진다.

```text
사용자들은 같은 주소로 요청
          ↓
     로드밸런서
  ├─ FastAPI A ─┐
  ├─ FastAPI B ─┼─ 공통 PostgreSQL 서버
  └─ FastAPI C ─┘          ↕
                      DB 데이터
```

> ➕ **더 알아두기 — “DB는 같아야 하잖아?”**
> A·B·C가 각각 다른 DB를 가지는 것이 아니라 같은 PostgreSQL 주소에 접속한다. A를 통해 저장한 글을 B도 같은 DB에서 읽는다. Hadoop 같은 분산 데이터 구조를 처음부터 쓴다는 뜻이 아니다. DB는 별도 컨테이너·서버 또는 관리형 DB 서비스일 수 있다.
>
> 여러 PostgreSQL 프로세스가 하나의 데이터 디렉터리 볼륨을 동시에 사용하는 것으로 이해하지 않는다. 여러 앱이 하나의 DB 서버에 연결하는 구조다. DB도 병목이 될 수 있어 쿼리·인덱스·자원·연결 관리 등을 따로 검토한다. 복제와 데이터 분할은 그 이후의 별도 설계다.
>
> 앱을 여러 개 실행하는 것 자체는 도커 없이도 가능하다. 도커는 동일한 실행 환경을 만들고 관리하는 도구다. 현재 실습은 backend 하나와 db 하나로, A·B·C 확장 구성을 구현하지 않았다.

## 주제 14: 이미지·라이브러리 변경과 DB 마이그레이션

> ➕ **더 알아두기**
> 라이브러리·프레임워크는 바꿀 수 있다. 다만 새 코드가 기존 DB 데이터·구조와 호환되는지 확인한다. 이미지에는 코드·실행 환경이, DB 볼륨에는 테이블 구조와 실제 데이터가 있다. 새 이미지가 DB를 자동 변환해 주지는 않는다.
>
> posts에 id·title·content만 있는데 새 코드가 author 열을 요구하면 기존 데이터를 유지하면서 열을 추가하는 마이그레이션(Migration)이 필요하다. FastAPI 버전만 바뀌고 테이블 사용 방식이 같다면 DB 변경이 필요하지 않을 수도 있다.
>
> 이전 앱 이미지로 되돌려도 DB 구조·데이터는 자동으로 되돌아가지 않는다. PostgreSQL 메이저 버전 업그레이드도 지원되는 별도 절차가 필요하다. 이번에는 PostgreSQL 17에서 pgvector가 추가된 17로 바꾼 것이며 이를 17→18에도 같은 볼륨을 무조건 바로 붙여도 된다고 일반화하면 안 된다.

> 📝 **기사 시험 포인트** (정보처리기사)
> 스키마(Schema, 데이터의 구조), 데이터 무결성(Data Integrity), 트랜잭션(Transaction)을 연결한다. 앱 배포와 DB 구조 변경은 별개이며 Git 커밋·DB COMMIT·스키마 마이그레이션도 서로 다른 작업이다.

## 주제 15: PostgreSQL을 벡터 DB로 사용하기

pgvector는 PostgreSQL에 벡터 저장·유사도 검색을 추가하는 확장 기능이다. 메모 같은 일반 데이터와 텍스트 임베딩을 같은 DB 인스턴스에서 관리할 수 있다. 별도의 벡터 DB 서버가 반드시 필요한 것은 아니다.

수업의 이미지를 적용하면서 서비스·버전 계열·경로·볼륨을 유지했다.

```yaml
image: pgvector/pgvector:pg17-trixie
```

```bash
docker compose up -d db
uv sync
uv run ingest.py
uv run retriever.py
```

Python에서는 langchain-postgres의 PGEngine·PGVectorStore를 사용한다. GoogleGenerativeAIEmbeddings가 Gemini API를 호출해 문서와 질문을 숫자로 바꾼다. 유효한 키가 필요하다.

현재 파일의 값은 TABLE_NAME=practice_texts_embedding2, VECTOR_SIZE=768, 모델 gemini-embedding-2다. 모든 벡터 DB에 공통인 고정값이 아니라 제공된 실습 코드의 선택이다.

| 고정 ID | 등록한 문장 주제 |
|---|---|
| image | 이미지의 파일·설정 |
| compose | 여러 컨테이너 실행 설정 관리 |
| volume | 컨테이너 재생성 후 데이터 유지 |
| fastapi | API·검증·자동 문서 |
| langgraph | 상태를 주고받는 그래프와 분기·반복 |
| react | 컴포넌트·상태에 따른 화면 갱신 |

ingest.py는 여섯 문장을 고정 ID로 넣는다. 같은 ID로 다시 실행할 때 문서가 계속 중복 증가하지 않게 구성되어 있다. retriever.py는 k=2로 관련 문서 두 개를 가져온다.

```text
등록: 문장 → Gemini 임베딩 → 원문과 숫자 768개를 PostgreSQL에 저장
검색: 질문 → Gemini 임베딩 → pgvector로 유사도 비교 → 관련 문장 2개
```

수업 질문으로 retriever.py와 POST /search를 실행한다.

```json
{
  "query": "컨테이너를 삭제해도 데이터를 유지하려면 어떻게 해야 하나요?"
}
```

> ➕ **더 알아두기 — 확인한 결과**
> 임베딩(Embedding)은 의미 비교에 쓰도록 텍스트를 숫자 벡터로 표현하는 과정이다. 실제 DB에서 vector 확장 버전 0.8.7, 문서 6개, 각 벡터 768차원을 확인했다. 기존 게시글 1개도 유지됐다. 위 질문은 volume 문서가 첫 번째, image가 두 번째였다. API 연속 두 번 호출도 성공했다.
>
> 다음 수업의 질문 “여러 컨테이너의 실행 설정을 한 파일로 관리하려면?”은 compose 문서가 첫 번째였다. 현재 API는 관련 자료를 찾아 반환한다. 이 자료로 생성 모델이 새 답변을 작성하는 RAG 생성 단계까지 구현한 것은 아니다. Gemini는 로컬 컨테이너에 들어 있는 모델이 아니라 외부 임베딩 API다.

## 주제 16: healthcheck·depends_on·성공 코드 0

DB 프로세스가 시작된 것과 연결을 받을 준비가 끝난 것은 다르다. 시작 때 DB 테이블을 준비하는 backend는 DB가 준비될 때까지 기다려야 한다.

```yaml
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d docker_prac"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 10s
```

| 옵션 | 의미 |
|---|---|
| test | 컨테이너 내부 검사 명령 |
| CMD-SHELL | 문자열을 컨테이너 셸에서 실행 |
| pg_isready | PostgreSQL 접속 준비 검사 |
| -U / -d | 검사 사용자 / 데이터베이스 |
| interval | 검사 간격 |
| timeout | 한 검사 제한 시간 |
| retries | 연속 실패 후 unhealthy로 표시할 횟수 |
| start_period | 초기 실패를 횟수에 넣지 않는 유예 기간 |

유예 기간에도 검사한다. 10초 전에 성공하면 바로 healthy가 되고 그 뒤 실패부터 센다. retries는 컨테이너 재시작 횟수가 아니다.

```yaml
    depends_on:
      db:
        condition: service_healthy
```

condition에는 Compose가 정한 service_healthy를 적는다. 종료 코드(Exit Code)는 검사 프로그램의 실행 결과 숫자이며 성공 코드 0을 Docker가 건강 상태로 해석한다.

```text
pg_isready 성공 → 종료 코드 0
       ↓
Docker가 db를 healthy로 판정
       ↓
service_healthy 조건 충족 → backend 시작
```

> ♻️ 처음엔 condition에 0을 넣으면 되는 것으로 알았으나, 0은 검사 명령의 성공 종료 코드이고 YAML의 조건값은 service_healthy라는 것이 맞다.

> ➕ **더 알아두기**
> YAML을 위에서부터 실행하는 것이 아니다. 서비스 의존 관계를 읽고 준비 상태를 기다린다. 검사는 FastAPI의 /health를 호출하는 것도 아니며 모든 테이블·인증·업무 기능 준비를 보장하지도 않는다.

시작 후 DB 검사는 계속되지만 depends_on만으로 실행 도중 DB 장애를 복구하거나 backend를 자동 재시작해 주지는 않는다.

## 주제 17: 실제 실습 경로와 Windows 호환 설정

> ➕ **더 알아두기 — 작업 대상 전환**
> 처음에는 board-practice에서 DB·health·볼륨을 실습했다. 사용자가 실제 수업 예제를 알려 주어 이후 다음 폴더에서 작업했다.

```text
<로컬 파일 경로>
```

> ➕ **더 알아두기 — 제공된 코드의 역할**
> main.py는 /health·/notes·/search와 시작 시 notes 테이블 생성을 담당한다. ingest.py는 문장 등록, retriever.py는 검색과 직접 실행 예제, vector_store.py는 공통 DB·임베딩·벡터 저장소 연결을 담당한다. pyproject.toml과 uv.lock은 의존성 요구 사항과 설치 버전을 관리한다.
>
> 상위 AGENTS.md에 따라 공용 <로컬 파일 경로> 사용하고 하위에는 새 .env를 만들지 않았다. 기존 Gemini 키를 사용했고 없던 DATABASE_URL을 추가했다. Python 코드는 load_dotenv(override=True)로 정리했다. 실제 키는 TIL에 기록하지 않는다.
>
> Mac과 Windows가 전체적으로 안 맞는다는 오류를 본 것이 아니다. Psycopg 비동기 연결은 Windows 기본 Proactor 대신 Selector 이벤트 루프가 필요하다. 기존 설정이 ingest.py에만 있어서 검색·API에도 적용되도록 vector_store.py로 옮겼다. 오류를 먼저 재현한 것은 아니며 코드에서 빠진 경로를 찾아 정리한 뒤 실제 등록·검색·API 호출을 검증했다.

```python
if sys.platform == "win32":
    asyncio.set_event_loop_policy(
        asyncio.WindowsSelectorEventLoopPolicy()
    )
```

> ➕ **더 알아두기**
> 오늘의 Python 3.13 실습에 맞춘 설정이다. Linux 컨테이너에서는 Windows 조건을 건너뛴다. 정책 API의 장기 버전 호환성까지 이번에 검증한 것은 아니다.

## 주제 18: 바인드 마운트·reload·uv·Uvicorn과 A/B

> ➕ **더 알아두기 — 먼저 필요한 개념**
> 파일은 저장된 코드이고 프로세스는 그 코드를 읽어 실행 중인 프로그램이다. 저장한다고 서버가 자동으로 다시 읽지는 않는다. reload는 변경을 감지해 서버 프로세스를 재시작한다. 파일 저장·서버 프로세스 재시작·컨테이너 재생성·브라우저 새로고침은 서로 다른 작업이다.

Dockerfile의 COPY는 빌드 당시 코드를 이미지에 담는다. 호스트 파일을 수정해도 기존 이미지의 코드는 그대로다. 기본 모드에서는 다시 빌드해 새 컨테이너로 적용한다.

바인드 마운트(Bind Mount)는 지정한 호스트 파일·폴더를 컨테이너 경로에 연결한다.

```yaml
volumes:
  - ./main.py:/app/main.py
```

연결 중에는 이미지의 해당 파일이 가려지고 호스트 파일이 보인다. 원본 이미지 파일을 삭제하거나 수정하는 것은 아니다.

| 구분 | 이름 있는 볼륨 | 바인드 마운트 |
|---|---|---|
| 대상 | Docker 관리 공간 | 지정한 호스트 파일·폴더 |
| 오늘 용도 | DB 데이터 보존 | main.py 개발 코드 연결 |
| 예 | postgres_data:/var/lib/postgresql/data | ./main.py:/app/main.py |

개발용 compose.dev.yaml은 기본 설정을 덮어쓴다.

```yaml
services:
  backend:
    command: ["uv", "run", "--no-sync", "fastapi", "dev", "main.py", "--host", "0.0.0.0", "--port", "8000"]
    environment:
      WATCHFILES_FORCE_POLLING: "true"
    volumes:
      - ./main.py:/app/main.py
```

command는 Dockerfile CMD 대신 실행한다. fastapi dev는 자동 reload를 켠다. polling은 파일 알림이 잘 전달되지 않는 환경에서도 주기적으로 변경을 확인한다.

```text
VS Code에서 main.py 저장
      ↓ 바인드 마운트
컨테이너에서 변경 파일이 보임
      ↓ WatchFiles / reload
서버 프로세스 재시작
      ↓
브라우저 새로고침으로 새 제목 확인
```

> ➕ **더 알아두기 — uv와 Uvicorn**
> uv는 패키지·환경 관리와 명령 실행 도구다. Uvicorn은 FastAPI 같은 ASGI 앱을 서비스하는 웹 서버다. uv run fastapi dev main.py는 uv로 FastAPI 개발 CLI를 실행하고 내부적으로 Uvicorn을 띄운다. uv run uvicorn main:app --reload처럼 직접 호출할 수도 있다. main:app은 main 모듈의 app 객체를 뜻한다.

> ♻️ 처음엔 바인드 마운트가 재시작 기능이라고 알았으나, 마운트는 파일을 연결하고 reload가 서버를 재시작하는 것이 맞다.

> ➕ **더 알아두기 — 이미지가 빈 껍데기는 아니다**
> 이미지에는 Python·라이브러리·시작 명령이 있으며 이번에는 앱 코드도 담는다. 바인드 마운트가 이미지 코드를 가릴 뿐이다. 코드 수정 때마다 빌드하지 않아도 되는 것이 개발상 이점이다. 컨테이너 단순 재시작만으로 이미지 코드가 바뀌지는 않는다.
>
> 같은 이미지의 운영 A에 마운트가 없으면 이미지 코드를 실행한다. 개발 B에 마운트가 있으면 호스트 파일을 실행한다. B를 껐다 켜도 마운트를 유지하면 수정된 호스트 코드가 다시 보인다. 마운트 없이 실행할 때 이미지 원본 파일이 보인다. 이번에는 A·B를 동시 실행한 것이 아니라 하나의 backend를 기본 모드에서 개발 모드로 바꿨다.
>
> 일반 배포에서는 개발 PC 소스 마운트·자동 reload 없이 검증된 이미지 코드를 사용한다. 모든 운영용 마운트가 금지라는 뜻으로 넓히지는 않는다. 현재 자동 반영 대상은 main.py 하나다.

## 주제 19: FastAPI 이미지·네트워크·기본/개발 모드 전체 실습

### 이미지 만들기와 빌드 제외 파일

제공된 Dockerfile을 사용했다. uv 버전은 해당 파일의 지정값으로 로컬 버전과 별개다.

```dockerfile
FROM python:3.13-slim

WORKDIR /app
RUN pip install --no-cache-dir uv==0.13.6
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev
COPY . .

CMD ["uv", "run", "--no-sync", "fastapi", "run", "main.py", "--host", "0.0.0.0", "--port", "8000"]
```

| 명령 | 역할 |
|---|---|
| FROM | Python이 있는 베이스 이미지 선택 |
| WORKDIR | 컨테이너 내부 기본 작업 위치 /app |
| 첫 RUN | 빌드 중 uv 설치 |
| 첫 COPY | 의존성 파일·잠금 파일 복사 |
| 둘째 RUN | 잠금 파일 기준 라이브러리 설치 |
| 둘째 COPY | 제외 목록 외 소스 복사 |
| CMD | 컨테이너 시작 시 실행할 기본 명령 |

RUN은 빌드 중 실행하고 CMD는 시작 명령을 저장한다. 의존성 파일을 먼저 복사해 코드만 바뀌면 설치 레이어를 캐시로 재사용한다. --frozen은 잠금 파일을 갱신하지 않고, --no-dev는 개발용 의존성을 제외하며, --no-sync는 실행 때 설치 환경 재동기화를 생략한다.

.dockerignore는 다음을 제외한다.

```text
.venv
.env
.env.*
__pycache__
*.pyc
.git
compose*.yaml
compose-examples/
```

Windows 가상환경과 API 키를 이미지에 복사하지 않는다. 필요한 Linux 라이브러리는 빌드 중 설치하고 키는 실행 시 전달한다. build: .의 점은 Compose 파일 폴더를 빌드 컨텍스트로 사용한다는 뜻이다.

### 실제 적용한 기본 Compose

수업의 ./.env 대신 상위 규칙의 공용 .env 경로를 사용했다.

```yaml
name: docker-practice

services:
  db:
    image: pgvector/pgvector:pg17-trixie
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: "<DB_PASSWORD>"
      POSTGRES_DB: docker_prac
    ports:
      - "127.0.0.1:5433:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d docker_prac"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 10s

  backend:
    build: .
    env_file:
      - ../../../../../.env
    environment:
      DATABASE_URL: postgresql://postgres:<DB_PASSWORD>@db:5432/docker_prac
    ports:
      - "127.0.0.1:8000:8000"
    depends_on:
      db:
        condition: service_healthy

volumes:
  postgres_data:
```

env_file은 파일의 환경변수를 컨테이너에 전달한다. environment는 직접 값을 지정하며 같은 키에 대해 우선한다. 따라서 공용 파일의 로컬 DB 주소를 고치지 않고 컨테이너에서 db:5432를 사용한다. Compose가 ${변수} 치환에 읽는 .env와 env_file의 역할은 구분한다.

> ➕ **더 알아두기**
> 이번 env_file은 공용 파일의 항목들을 backend 환경에 전달한다. 키를 이미지에 굽지 않았다는 것이 실행 환경에도 키가 없다는 뜻은 아니다. 공용 파일을 쓰는 규칙과 예제의 ./env 경로 차이를 반영한 설정이다. 실제 키 문자열은 기록에 넣지 않는다.

### 컨테이너 사이의 주소와 0.0.0.0

| FastAPI 위치 | DB 주소 |
|---|---|
| Windows 로컬 | 127.0.0.1:5433 |
| backend 컨테이너 | db:5432 |

```text
브라우저 → 호스트 127.0.0.1:8000
                ↓ 포트 전달
          backend 컨테이너 :8000
                ↓ db:5432
          DB 컨테이너 :5432
                ↕
          postgres_data 볼륨
```

Compose 기본 네트워크에서는 서비스 이름 db로 DB를 찾는다. backend의 localhost는 backend 자신이다. 호스트에 공개한 5433이 아니라 DB 내부 5432로 접속한다.

FastAPI의 --host 0.0.0.0은 해당 컨테이너의 모든 네트워크 인터페이스에서 받겠다는 뜻이다. 내부 127.0.0.1에만 바인딩하면 외부에서 전달된 요청을 받을 수 없다. 브라우저에는 0.0.0.0 대신 localhost나 127.0.0.1을 입력한다. ports의 앞쪽 127.0.0.1은 공개 범위를 내 컴퓨터로 제한한다.

### 실제 실행·변경 반영·검증 결과

8000 포트를 사용하던 로컬 FastAPI를 종료한 뒤 이미지 빌드와 기본 컨테이너 실행을 진행했다.

```bash
docker compose config -q
docker compose up -d --build
docker compose ps
docker compose logs backend
```

config -q는 유효성 검사로 성공하면 출력이 없다. 실제 작업에서는 build backend와 up을 나누어 실행하기도 했다. DB healthcheck가 성공한 뒤 backend가 시작되는 로그를 확인했다.

1. 기본 모드의 /health에서 db:5432 연결을 확인했다.
2. 기존 /notes 메모를 조회했고 /search에서 Compose 문서가 첫 번째로 나왔다.
3. main.py 제목을 “나의 Compose 실습”으로 바꿔 up -d --build backend를 실행했고 OpenAPI 제목 변경을 확인했다.
4. 개발용 설정을 적용했다.

```bash
docker compose -f compose.yaml -f compose.dev.yaml config -q
docker compose -f compose.yaml -f compose.dev.yaml up -d
```

5. 준비 후 제목을 “Bind mount 개발 실습”으로 수정·저장했다.
6. build나 up을 다시 실행하지 않고 같은 컨테이너 ID·이미지 ID에서 제목이 바뀐 것을 확인했다. 로그에 WatchFiles 변경 감지와 기존 서버 프로세스 종료·새 프로세스 시작이 남았다.
7. 개발 모드에서도 /health·/notes·/search가 정상 응답했다.

compose.dev.yaml은 파일이 존재한다고 자동 적용되지 않는다. -f로 기본 파일 뒤에 개발용 파일을 명시한다. command·마운트가 달라지면 컨테이너는 재생성된다. 그 뒤 main.py 저장에 따른 reload는 컨테이너 재생성과 다르다.

```bash
# 실습 폴더로 이동
cd <로컬 파일 경로>

# 개발 모드
docker compose -f compose.yaml -f compose.dev.yaml up -d

# 의존성·Dockerfile·마운트하지 않은 소스 변경 후 개발용 재빌드
docker compose -f compose.yaml -f compose.dev.yaml up -d --build

# 이미지에 담긴 코드로 기본 모드 전환
docker compose up -d

# 현재 코드를 이미지에도 반영해 기본 모드 실행
docker compose up -d --build

# 컨테이너를 남기고 중지
docker compose stop
```

완료 시점에는 backend·db가 실행 중이고 DB는 healthy였다. 외부 클라우드 배포는 하지 않았다. 주소는 http://127.0.0.1:8000/docs이며, 현재 상태는 이후 중지·설정 변경에 따라 달라질 수 있다.

## 주제 20: 수업 출처와 학습 방식

수업 번호는 숫자순이 아니라 아래 순서다. 첫 링크는 마지막 레슨이었고 사용자가 순서를 정정했다. 여섯 페이지 본문을 내부 브라우저에서 직접 확인했다. 일반 웹 도구와 외부 브라우저 제어 실패를 “수업을 읽을 수 없음”으로 단정하지 않았다.

1. [1472 — 도커 기초](https://ucanlabs.kr/classroom/aicampus-agent-python/lesson/1472/)
2. [1473 — 도커로 PostgreSQL 실행하기](https://ucanlabs.kr/classroom/aicampus-agent-python/lesson/1473/)
3. [1477 — 로컬 서버에서 도커 DB 연결하기](https://ucanlabs.kr/classroom/aicampus-agent-python/lesson/1477/)
4. [1474 — 볼륨으로 데이터 유지하기](https://ucanlabs.kr/classroom/aicampus-agent-python/lesson/1474/)
5. [1475 — PostgreSQL을 벡터 DB로 사용하기](https://ucanlabs.kr/classroom/aicampus-agent-python/lesson/1475/)
6. [1476 — 도커에서 FastAPI 실행하기](https://ucanlabs.kr/classroom/aicampus-agent-python/lesson/1476/)

일반 본문은 읽은 레슨을 바탕으로 한다. ➕는 대화 보충·실제 프로젝트 적용 기록, ♻️는 이해를 바로잡은 지점, 📝는 기사 대비 연결 개념이다. 기사 블록이 강사가 오늘 시험 범위라고 직접 말했다는 뜻은 아니다.

> ➕ **더 알아두기 — 앞으로의 설명 방식**
> 사용자는 중간 단계를 건너뛰고 질문할 수 있으므로 이해에 필요한 선행 정보도 설명해 달라고 요청했다. 도구 이름을 나열하기보다 코드 파일·프로세스·이미지·컨테이너·볼륨·커널의 층위를 먼저 나눈다. 특히 바인드 마운트에서는 파일 저장과 실행 프로그램 재시작의 차이를 먼저 설명해야 이해가 쉬웠다. 명령은 사용자의 Git Bash 문법으로 안내한다.

## 정리

Docker는 이미지로 실행 환경을 준비하고 Engine이 컨테이너를 실행·관리한다. Desktop은 실행 기반·도구·화면을 제공하고 Hub는 이미지 레지스트리다. WSL 2·Ubuntu·Docker 전용 배포판·컨테이너 이미지를 구분한다.

실습 DB는 pgvector가 포함된 PostgreSQL 17이며 이름 있는 볼륨으로 데이터를 유지한다. 로컬 FastAPI는 5433, 컨테이너 FastAPI는 db:5432로 연결한다. Gemini가 문장·질문을 벡터로 만들고 PostgreSQL이 저장·비교한다. 현재 구현은 관련 문장 검색까지다.

기본 모드는 이미지 코드, 개발 모드는 마운트한 main.py와 reload를 사용한다. 파일 연결·서버 재시작·컨테이너 재생성·재빌드는 다르다. 재시작해도 마운트가 유지되면 호스트 코드를 쓴다. 도커는 배포·개발 관리를 돕지만 DB 마이그레이션·백업·보안·성능을 자동 해결하지 않는다.

## ✅ 확인 질문

1. Docker가 코드만 전달하는 방식에 비해 줄이는 환경 준비 문제를 세 가지 설명하라.
2. 노트북과 클라우드에서 컨테이너를 실행할 때 실제 자원을 제공하는 컴퓨터는 무엇인가?
3. VM과 컨테이너에서 커널·사용자 공간을 어떻게 다르게 취급하는가?
4. .venv와 컨테이너의 격리 범위와 함께 쓸 수 있는 이유를 설명하라.
5. Engine·CLI·Desktop의 포함 관계와 요청 흐름을 각각 설명하라.
6. Desktop 화면을 프론트엔드, Engine을 백엔드로 비유할 때의 한계는 무엇인가?
7. Hub와 Desktop의 차이와 이미지 태그가 내용의 영구 고정을 보장하지 않는 이유를 설명하라.
8. 호스트 폴더를 악성 컨테이너에 쓰기 가능으로 연결하면 어떤 영향이 있는가?
9. WSL 2·Ubuntu·docker-desktop·컨테이너 이미지를 층위별로 설명하라.
10. 현재 확인한 Docker 전용 배포판 정보와 확인하지 못한 내부 기반 정보는 무엇인가?
11. Dockerfile과 Compose는 각각 무엇을 정하며 여러 서비스는 이미지 하나로 합쳐지는가?
12. 프로젝트·서비스·image·YAML 들여쓰기의 역할을 설명하라.
13. 127.0.0.1:5433:5432를 해석하고 기존 로컬 DB와 충돌하지 않는 이유를 설명하라.
14. POSTGRES_PASSWORD의 초기화 적용과 기존 DB 비밀번호 변경은 어떻게 다른가?
15. GET /posts가 200이어도 도커 DB 연결을 확정할 수 없는 이유는 무엇인가?
16. /health가 설정과 실제 DB 조회 결과를 함께 보여 주는 이유는 무엇인가?
17. 볼륨 연결 범위·특정 테이블 선택·용량 제한은 어떻게 다른가?
18. 비어 있는 postgres_data:와 실제 데이터가 없다는 말은 왜 다른가?
19. stop/start와 down/up을 컨테이너 존재·데이터 보존 관점에서 비교하라.
20. 익명 볼륨과 이름 있는 볼륨은 컨테이너 재생성 뒤 연결에서 어떤 차이가 있는가?
21. 동일 이미지 컨테이너를 여러 개 만들면 디스크가 무조건 n배가 아닌 이유는 무엇인가?
22. 이미지 공유가 CPU·메모리·볼륨·로그 비용까지 없애지 않는 이유는 무엇인가?
23. 환경이 준비된 서버에서도 도커가 유용한 상황과 도입 이점이 작은 상황을 설명하라.
24. 사용자 경험과 도커 사용 여부, 운영 품질의 간접 영향을 구분하라.
25. A·B·C 앱이 같은 게시판 데이터를 보도록 DB 접속을 어떻게 설정하는가?
26. 여러 앱의 공통 DB 접속과 여러 PostgreSQL의 같은 데이터 디렉터리 공유는 왜 다른가?
27. 앱 이미지 교체와 DB 스키마 마이그레이션은 왜 별도 작업인가?
28. 이전 이미지로 앱을 되돌리면 DB도 자동 복원되지 않는 이유를 설명하라.
29. Gemini 임베딩·pgvector·PGVectorStore·Retriever의 역할과 검색 흐름을 설명하라.
30. 관련 문장 두 개 반환과 RAG 답변 생성의 차이는 무엇인가?
31. pg_isready 종료 코드 0과 condition: service_healthy의 관계를 설명하라.
32. YAML 순서·DB 시작·DB 준비 완료·시작 후 장애 복구를 구분하라.
33. Windows 이벤트 루프 설정을 공통 파일로 옮긴 이유와 실제 오류 재현 여부를 설명하라.
34. 실제 .env 위치와 로컬용·컨테이너용 DB 주소를 설명하라.
35. 파일 저장·바인드 마운트·reload·브라우저 새로고침의 역할을 구분하라.
36. 운영 A와 개발 B의 같은 이미지에서 B를 재시작하거나 마운트를 제거하면 어떤 코드가 보이는가?
37. RUN·COPY·CMD의 실행 시점과 의존성 파일을 먼저 복사하는 이유를 설명하라.
38. 컨테이너 내부 0.0.0.0:8000, 호스트 127.0.0.1:8000, db:5432를 구분하라.
39. 기본/개발 Compose 실행·재빌드 명령과 자동 반영 파일 범위를 설명하라.
40. 오늘 직접 검증한 결과와 아직 구현하지 않은 운영 구성·확인하지 못한 내용을 구분하라.
