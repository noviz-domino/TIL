---
tags: [rds, pgvector, tls, vscode, deployment]
til: v2 2026-11-02
---

# RDS 접속·pgvector 활성화·VS Code 연결과 백엔드 업로드
> 작성일: 2026-11-02

## 🔗 관련 글

- [AWS 배포 기초와 RDS·EC2·Docker 준비](2026-11-02%20(2)AWS_배포_기초와_RDS_EC2_Docker_준비.md) — RDS·EC2 생성, 두 출발지를 허용하는 보안 그룹, Docker 설치를 마친 뒤 실제 DB 접속으로 이어진다.
- [pgvector 전환 코드 복습](2026-11-02%20(1)pgvector_전환코드_복습(패키지,_설정,_임베딩차원,_ORM모델).md) — PostgreSQL의 벡터 확장과 임베딩 저장 개념을 실제 RDS에서 확인한다. 서로 다른 실습 코드의 경로와 테이블 이름까지 같다고 가정하지 않는다.
- [Docker와 Compose 실습](2026-10-30%20(1)Docker와_Compose_실습(WSL2,_볼륨,_pgvector,_FastAPI,_바인드마운트).md) — 로컬 DB를 함께 실행하던 Compose를 외부 RDS에 접속하는 배포용 Compose와 비교한다.

## ❓ 아직 모르겠는 것

대화에서 설명을 들었거나 실습으로 해결한 질문도 복습 때 다시 확인한다. 확인하지 않은 작업은 미해결로 표시한다.

- “엔드포인트는 찾았는데 무엇을 입력해야 하나?” — EC2에서 psql 클라이언트 컨테이너를 실행하는 명령과 접속 인자를 구분하는가? (주제 1)
- “VS Code에서도 쓸 수 있나? SSL을 켜고 CA certificate에 컴퓨터에 받은 PEM을 연결하라는데?” — 접속 도구·서버 위치·인증서 파일 위치를 구분하는가? (주제 3)
- “Git Bash에서 VS Code를 켜는 코드가 뭐였지?” — code .에서 점의 의미는 무엇인가? (주제 3)
- “VS Code를 열었는데 Connect가 어디 있지?”, “SSL 토글을 켰는데 여기서 무엇을 하나?” — DB 확장의 연결 화면에서 각 항목에 무엇을 넣는가? (주제 3)
- “설정했는데 된 건가?” — 입력 완료와 실제 접속 성공을 어떤 화면·SQL 결과로 구분하는가? (주제 3·4)
- 사용자가 연결 화면과 오류 로그를 공유했다. — Connection Success와 rdsadmin 접근 거절이 동시에 나오는 이유는 무엇인가? (주제 4)
- “백엔드 배포할 차례인가?” — DB 연결 확인과 FastAPI 배포는 왜 다른 작업인가? (주제 5·7)
- “이 backend 폴더의 YAML과 코드가 맞나? GitHub private 레포를 만들어 올려달라.” — 원본 로컬 설정에서 무엇을 바꿨고 어떤 파일을 업로드했는가? (주제 5)
- “아까 인바운드는 왜 두 개라고 했지?” — 로컬 PC와 EC2의 접속 허용을 구분하는가? (주제 6)
- “EC2에서 지정해서 EC2 보안 그룹인 줄 알았다.” — 화면이 있는 서비스 메뉴와 실제 규칙 적용 대상을 구분하는가? (주제 6)
- EC2로 private 저장소를 가져와 FastAPI를 실행했는가? — GitHub push까지만 완료했으며 EC2 clone·환경변수 설정·기동은 아직 진행하지 않았다. (주제 7, 미해결)
- 검색 문서를 임베딩해서 RDS에 적재했는가? — vector 확장 활성화만 확인했으며 ingest.py 실행과 검색 결과는 아직 확인하지 않았다. (주제 2·7, 미해결)

## 주제 1: EC2에서 RDS PostgreSQL에 실제 접속

### CA 인증서 준비와 접속 명령

앞선 실습에서 RDS의 엔드포인트, 즉 DB 접속 주소를 확인했다. 교안에 따라 EC2 Ubuntu 터미널에서 공개 CA 인증서 묶음을 준비하고, PostgreSQL 이미지의 임시 컨테이너에서 psql을 실행했다. 아래 접속 명령은 실제 주소를 공개용 자리표시자로 바꾼 재현 예시다. 실제 암호나 자원 주소는 기록하지 않는다.

실행 위치는 AWS 브라우저로 연결한 EC2 Ubuntu 터미널이다.

```bash
curl -fsSL https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem -o /home/ubuntu/global-bundle.pem
```

```bash
sudo docker run --rm -it \
  -v /home/ubuntu/global-bundle.pem:/certs/global-bundle.pem:ro \
  postgres:17-alpine \
  psql "host=<RDS_ENDPOINT> port=5432 dbname=aim_agent user=postgres sslmode=verify-full sslrootcert=/certs/global-bundle.pem"
```

| 부분 | 역할 |
|---|---|
| sudo docker run | 관리자 권한으로 Docker 컨테이너 실행 |
| --rm | 프로세스가 끝나면 해당 컨테이너 정리 |
| -it | 사용자가 터미널에서 대화식으로 입력 가능하게 실행 |
| -v 호스트경로:컨테이너경로:ro | EC2의 CA 파일을 컨테이너 내부에 읽기 전용으로 연결 |
| postgres:17-alpine | PostgreSQL 17 계열 도구가 들어 있는 이미지 |
| psql | PostgreSQL에 접속하고 SQL을 입력하는 클라이언트 프로그램 |
| host | RDS 엔드포인트 |
| port=5432 | PostgreSQL 접속 포트 |
| dbname=aim_agent | 접속할 실제 데이터베이스 이름 |
| user=postgres | DB 로그인 계정 |
| sslmode=verify-full | TLS 암호화와 서버 인증서·호스트 이름 검증 |
| sslrootcert | 컨테이너 안에서 읽을 CA 인증서 파일 위치 |

> ➕ **더 알아두기 — 클라이언트 컨테이너와 DB 서버**
> 이 명령은 EC2 안에 별도 PostgreSQL 서버를 띄워 데이터를 저장하는 명령이 아니다. 이미지에 들어 있는 psql을 실행해서 이미 생성한 RDS 서버에 접속한다. 데이터는 계속 RDS에 있고, 접속 도구만 EC2의 임시 컨테이너에서 실행되는 구조다.
>
> 파일 마운트의 왼쪽은 EC2의 파일 위치, 오른쪽은 컨테이너 내부 위치다. EC2에서 파일이 존재해도 컨테이너에서 자동으로 같은 경로를 볼 수 있는 것은 아니다. 명령의 sslrootcert는 오른쪽 컨테이너 경로를 사용한다.

### 실제 결과 확인

처음 실행할 때 postgres:17-alpine 이미지가 EC2에 없다는 메시지 뒤로 이미지 다운로드가 진행됐다. 이후 postgres 사용자의 암호를 묻는 화면에서 RDS 생성 시 설정한 마스터 암호를 입력했다. 암호는 화면에 표시되지 않으며 채팅이나 기록에 넣지 않는다.

실제 캡처에서 아래를 확인했다.

```text
psql (17.11)
SSL connection (protocol: TLSv1.3, ...)
aim_agent=>
```

TLSv1.3 연결 안내와 aim_agent 프롬프트는 DB에 암호화된 연결로 로그인한 결과다. 앞선 TIL에서는 CA 파일 다운로드 결과와 RDS 로그인 여부가 미확인이었지만, 이번에는 verify-full로 접속한 성공 결과를 확인했다. 파일을 cat으로 열어 내용 자체를 검사한 것은 아니다.

> ➕ **더 알아두기 — 프롬프트 변화**
> ubuntu@...:~$에서는 Ubuntu 셸 명령을 입력하고, aim_agent=>에서는 SQL 또는 psql 전용 명령을 입력한다. 같은 터미널 창이라도 현재 어떤 프로그램 안에 있는지에 따라 입력할 명령이 달라진다. 줄이 이어질 때 보이는 aim_agent->는 SQL 문장이 아직 끝나지 않았음을 나타낼 수 있으므로 세미콜론까지 입력한다.

## 주제 2: pgvector 활성화와 데이터 적재의 차이

교안은 RDS에 연결된 psql에서 vector 확장을 활성화하고 이름·버전을 조회하도록 한다. 확장(Extension)은 PostgreSQL에 추가 기능을 제공하는 구성 요소다. pgvector는 벡터 저장과 유사도 검색을 위한 확장이다.

실제 실행한 SQL은 다음과 같다.

```sql
CREATE EXTENSION IF NOT EXISTS vector;

SELECT extname, extversion
FROM pg_extension
WHERE extname = 'vector';
```

첫 명령은 현재 접속한 DB에서 vector 확장을 활성화한다. IF NOT EXISTS는 이미 존재할 때 중복 생성 오류가 발생하지 않도록 한다. 두 번째 SQL은 설치된 확장 정보를 담는 시스템 카탈로그 pg_extension에서 vector 행을 조회한다.

실제 결과는 다음과 같았다.

```text
CREATE EXTENSION

 extname | extversion
---------+------------
 vector  | 0.8.2
(1 row)
```

aim_agent DB에서 pgvector 0.8.2 활성화가 확인됐다. 이 확장 활성화의 적용 대상은 현재 접속한 데이터베이스다.

> ➕ **더 알아두기 — 기능 준비와 실제 데이터**
> vector 확장을 활성화했다고 검색할 문서가 들어간 것은 아니다. 벡터를 다룰 기능을 준비한 것이다. 다음 작업으로 코드에서 필요한 테이블을 준비하고, 문서를 임베딩, 즉 숫자 벡터로 변환해서 저장해야 한다. 교안의 ingest.py는 검색할 문서를 DB에 적재하는 단계이며, 아직 실행 결과를 확인하지 않았다.
>
> DB 서버 생성, DB 로그인, 확장 활성화, 테이블 준비, 데이터 적재, 검색은 각각 다른 작업이다. 확장 확인 결과 한 줄만으로 전체 검색 서비스가 완성됐다고 판단하지 않는다.

이후 DB 접속을 종료하는 psql 명령도 안내했다.

```text
\q
```

> ➕ **더 알아두기 — 접속 종료와 삭제**
> \q는 psql을 종료하는 명령이며 DB나 확장을 삭제하지 않는다. 이 방식에서는 psql이 끝나면 --rm으로 임시 컨테이너도 정리된다. RDS는 계속 존재한다. 사용자가 실제로 \q를 실행한 결과는 따로 확인하지 않았다.

## 주제 3: 로컬 PC의 VS Code에서 같은 RDS 연결

### EC2 밖에서도 접속할 수 있는가

수업에서는 VS Code의 DB 확장 프로그램으로 RDS에 접속했다. 실행 위치는 사용자 PC이며 RDS는 계속 AWS에 있다. EC2 터미널과 VS Code는 서로 다른 접속 도구지만 같은 엔드포인트·DB 이름을 사용하면 같은 DB를 볼 수 있다.

> ➕ **더 알아두기 — 접속하는 곳과 저장되는 곳**
> EC2의 psql → RDS, 로컬 PC의 VS Code → RDS라는 두 경로를 구분한다. VS Code에서 DB 내용을 조회한다고 RDS 데이터가 로컬 PostgreSQL로 옮겨지는 것은 아니다. DB 서버와 클라이언트 프로그램은 별개다.

### Git Bash에서 VS Code 열기

사용자 PC의 Git Bash에서 현재 폴더를 VS Code로 여는 명령은 다음과 같다.

```bash
code .
```

> ➕ **더 알아두기 — 점의 의미**
> .은 현재 디렉터리다. 이 명령은 현재 폴더를 편집기로 열 뿐이며 AWS 접속이나 DB 로그인까지 해주는 명령은 아니다. 이번 화면의 DB 연결 기능은 별도 DB 확장 프로그램이 제공한다.

### 연결 화면에 넣은 값

VS Code에서 PostgreSQL을 선택하고 SSL 토글을 켰다. 실제 연결 화면의 항목은 다음과 같이 채웠다. 주소와 암호는 공개 기록에서 제외한다.

| 항목 | 설정·의미 |
|---|---|
| Name | AWS 실습 RDS — 연결을 알아보기 위한 표시 이름 |
| Group | AWS — 연결 목록을 묶는 폴더·그룹 |
| Server Type | PostgreSQL |
| Host | <RDS_ENDPOINT> — localhost나 127.0.0.1이 아닌 RDS 주소 |
| Port | 5432 |
| Username | postgres |
| Password | RDS 마스터 암호, 기록하지 않음 |
| Database | aim_agent — 기본 표시 postgres에서 변경 |
| SSL | 켬 |
| SSL Config / CA Certificate | 로컬 PC의 global-bundle.pem 선택 |

SSL 설정 이름을 사용하더라도 실제 통신 보호에는 TLS를 사용한다. CA 인증서 파일은 클라이언트가 RDS 서버 인증서를 검증할 때 사용한다. 강사 안내와 대화에서 이 파일을 지정하도록 했고 사용자는 설정했다고 답했다. 다만 SSL Config 세부 화면 자체는 공유되지 않았으므로 해당 확장의 구체적인 검증 옵션을 모두 확인했다고 기록하지 않는다.

로컬 PC에도 별도 인증서 파일이 필요하므로 다음 다운로드 명령을 안내했다. 실행 위치는 로컬 PC의 Git Bash다.

```bash
curl -fsSL https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem -o global-bundle.pem
```

> ➕ **더 알아두기 — 같은 이름의 파일, 다른 컴퓨터**
> EC2에 받은 파일은 EC2에만 있다. VS Code에서 사용하는 파일은 로컬 PC에서 접근 가능한 위치에 있어야 한다. 상대 경로로 다운로드했다면 당시 Git Bash의 현재 폴더에 저장된다. 실제 CA 파일은 공개 인증서 묶음이며 비밀 키가 아니다. 확장자가 .pem이라고 해서 모두 비밀 키인 것은 아니다.

연결 설정을 입력한 다음 Connect를 눌렀다. 화면에 Connection Success!가 나타났고, 왼쪽 AWS 그룹을 펼치자 AWS 실습 RDS 아래에 aim_agent DB가 보였다. 기존 로컬 연결은 127.0.0.1@5432로 별도 표시돼 있었다.

> ➕ **더 알아두기 — 연결 이름과 대상 검증**
> 연결 이름은 사용자가 정할 수 있는 표시 문자열이므로 이름만 보고 대상이 맞다고 판단하지 않는다. Host와 Database를 확인하고, 실제 SQL로 현재 DB를 조회한다. 설정 입력 완료, 연결 성공 메시지, 실제 SQL 조회 성공은 확인 수준이 서로 다르다.

> 📝 **기사 시험 포인트** (정보처리기사·정보보안기사)
> 클라이언트·서버 구조에서 같은 서버에 여러 클라이언트가 접속할 수 있다. TLS의 암호화는 통신 내용 보호에, 인증서 검증은 서버 신원 확인에 관계된다. CA(Certificate Authority, 인증 기관)와 개인 키의 역할을 구분한다. 인증서를 지정하는 것과 검증 옵션이 실제로 적용됐는지 확인하는 것도 구분한다.

## 주제 4: rdsadmin 오류와 실제 연결 검증

### 성공 메시지가 있는데 왜 빨간 오류가 나오는가

VS Code에서 Connection Success!가 표시되고 aim_agent 목록도 보였지만, 아래 Output 로그에 pg_hba.conf rejects connection이라는 오류가 나타났다. 실제 로그의 대상 DB는 aim_agent가 아니라 rdsadmin이었다. 공개 기록에서는 로그 안의 원본 IP를 제외한다.

```text
pg_hba.conf rejects connection for host "<CLIENT_IP>", user "postgres",
database "rdsadmin", SSL encryption
```

> ➕ **더 알아두기 — 오류의 대상을 먼저 읽기**
> rdsadmin은 AWS가 RDS 운영을 위해 사용하는 내부 관리 영역이다. 일반 실습 사용자가 접속해서 작업할 DB가 아니다. 확장이 여러 DB의 메타데이터, 즉 구조 정보를 읽으려다 이 관리용 DB에도 접속을 시도한 것으로 보였다. 이 설명은 rdsadmin 트리를 만드는 로그와 거절된 DB 이름에 근거한 해석이다.
>
> 이번 로그의 끝에 SSL encryption이 있다는 이유만으로 SSL 설정을 끄거나 보안 그룹을 넓힐 필요는 없다. 어떤 DB·사용자·접속이 거절됐는지 먼저 확인한다. 여기서는 실습 DB인 aim_agent가 정상 조회되는지 별도로 검증했다.

### VS Code에서 실제로 실행한 SQL

왼쪽 aim_agent 아래 Query를 열고 다음 SQL을 실행했다. 로컬 DB나 rdsadmin 아래 Query를 선택하지 않도록 구분했다.

```sql
SELECT current_database(), extname, extversion
FROM pg_extension
WHERE extname = 'vector';
```

| 조회식·컬럼 | 의미 |
|---|---|
| current_database() | 현재 접속한 DB 이름 |
| extname | 확장 이름 |
| extversion | 확장 버전 |
| WHERE extname = 'vector' | vector 확장 행만 조회 |

실제 VS Code 결과 표에 다음 한 행이 표시됐다.

```text
current_database | extname | extversion
aim_agent        | vector  | 0.8.2
```

이 결과로 로컬 PC의 VS Code에서 RDS의 aim_agent에 SQL을 실행했고, EC2에서 켠 vector 0.8.2를 확인했다. 성공 메시지만 본 상태에서 실제 데이터 조회까지 확인한 상태로 진행했다.

> ➕ **더 알아두기 — 해결 범위**
> rdsadmin의 접근 제한을 해제한 것이 아니다. 우리가 사용할 aim_agent의 조회 성공을 확인하고 관리용 DB 오류와 구분한 것이다. AWS가 관리하는 DB의 권한을 바꾸는 작업은 하지 않았다.

## 주제 5: 배포용 백엔드 확인·별도 준비·GitHub 업로드

### 어떤 폴더를 확인했는가

사용자가 지정한 실습 프로젝트의 docker/docker_prac/backend 폴더를 확인했다. main.py, vector_store.py, retriever.py, ingest.py, Dockerfile, pyproject.toml, uv.lock, compose.yaml 등이 있어 이번 배포에 사용할 백엔드 폴더임을 확인했다. 개인 PC의 전체 절대 경로는 공개 기록에 남기지 않는다.

원본 main.py에는 /health, /notes, /search API가 있고 시작 시 notes 테이블이 없으면 생성하는 코드가 있었다. ingest.py에는 Docker·Compose·볼륨·FastAPI·LangGraph·React에 관한 예제 문서 6개가 있었다. vector_store.py는 DB 연결, 벡터 테이블 준비, 임베딩 객체 생성을 담당하고 retriever.py는 검색을 호출한다.

원본 파일을 직접 배포용으로 덮어쓰지 않고, 별도 aws-deploy-backend 폴더에 필요한 파일을 복사해 준비했다.

### 로컬 실습 Compose와 RDS 배포 Compose의 차이

원본 compose.yaml은 로컬 실습용이었다. db 서비스에서 pgvector 이미지를 실행하고, 이름 있는 볼륨에 데이터를 저장하며, backend가 Docker 네트워크의 db 호스트로 접속하는 구성이었다. 원본은 상위 폴더의 공용 .env를 읽는 경로도 사용했다.

> ➕ **더 알아두기 — 그대로 복사하면 안 되는 부분**
> db는 원본 Compose 안에서 실행되는 로컬 DB 서비스의 이름이다. RDS 엔드포인트가 아니다. EC2 배포에서는 이미 준비한 RDS에 접속해야 하므로 로컬 DB 서비스를 그대로 띄우는 대신 백엔드의 연결 대상을 외부 RDS로 바꾼다. 로컬 프로젝트의 상위 폴더에 있던 공용 .env 경로도 EC2에서 그대로 존재한다고 가정할 수 없다.

배포용 폴더에 저장하고 push한 Compose는 다음과 같다. RDS는 외부 서비스이므로 Compose 안에서 db 컨테이너를 만들지 않는다.

```yaml
name: deploy-backend

services:
  backend:
    build: .
    env_file:
      - ./.env
    environment:
      PGSSLMODE: verify-full
      PGSSLROOTCERT: /certs/global-bundle.pem
    volumes:
      - /home/ubuntu/global-bundle.pem:/certs/global-bundle.pem:ro
    ports:
      - "127.0.0.1:8000:8000"
    restart: unless-stopped
```

| 설정 | 의미 |
|---|---|
| build: . | 현재 폴더의 Dockerfile로 백엔드 이미지 빌드 |
| env_file: ./.env | 배포용 폴더에 별도로 준비할 실제 환경변수 파일 읽기 |
| PGSSLMODE: verify-full | RDS TLS 연결의 서버 인증서와 호스트 이름 검증 |
| PGSSLROOTCERT | 컨테이너에서 읽을 CA 인증서 위치 |
| volumes의 :ro | EC2의 CA 파일을 읽기 전용으로 연결 |
| 127.0.0.1:8000:8000 | EC2의 루프백 주소 8000번 포트와 컨테이너 포트 연결 |
| restart: unless-stopped | 명시적으로 중지한 경우를 제외하고 재시작 정책 적용 |

> ➕ **더 알아두기 — 공개 포트와 환경변수**
> 127.0.0.1은 그 컴퓨터 자신을 가리키는 루프백 주소다. 이 설정은 EC2 내부에서 먼저 /health를 확인하고 이후 Nginx를 붙이기 위한 구성이다. 인터넷 사용자가 EC2의 8000번 포트로 바로 접속하는 것을 목표로 한 설정이 아니다.
>
> .env는 실행 환경의 실제 값을 담고, .env.example은 어떤 항목을 채워야 하는지 안내하는 예제다. 예제에 자리표시자가 있다고 실제 비밀 값이 자동으로 채워지는 것은 아니다. RDS 암호는 URL에 넣지 않고 PGPASSWORD로 전달하도록 예제를 준비했다.

저장한 .env.example의 내용은 다음과 같다. 실제 값은 없다.

```dotenv
DATABASE_URL=postgresql://postgres@<RDS_ENDPOINT>:5432/aim_agent
PGPASSWORD='<DB_PASSWORD>'
GEMINI_API_KEY=<GEMINI_API_KEY>
```

### 업로드 파일과 제외한 파일

비공개 GitHub 저장소 aws-agent-backend를 만들었다. 커밋 전에 포함 파일과 메시지를 보여주고 사용자 승인을 받았다. 승인한 파일 12개만 커밋한 뒤 main 브랜치에 push했다.

```text
.dockerignore
.env.example
.gitignore
Dockerfile
README.md
compose.yaml
main.py
ingest.py
retriever.py
vector_store.py
pyproject.toml
uv.lock
```

.gitignore에는 실제 .env와 변형 파일, .venv, __pycache__, Python 캐시, *.pem, *.key, 편집기 설정 폴더를 제외하도록 작성했다. .env.example만 예외로 포함했다. 원본의 compose.dev.yaml과 compose-examples 폴더는 배포용 복사본에 포함하지 않았다. README에는 EC2 실행 준비·환경변수·기동·검색 데이터 적재의 안내를 담았다.

> ➕ **더 알아두기 — private와 비밀 정보 관리**
> private 저장소라도 실제 암호와 API 키를 소스에 넣지 않는다. 비공개 설정은 접근할 수 있는 사람을 제한하는 것이며, 비밀 값을 Git 이력에 남겨도 된다는 뜻이 아니다. 이번 업로드에는 실제 .env·인증서·암호·API 키·RDS 주소를 넣지 않았다. 정해진 파일만 포함하고 자격 증명 패턴도 검사했다. 패턴 검사 하나만으로 모든 종류의 비밀을 완벽하게 찾는다고 보장하는 것은 아니다.

### 확인한 검사와 push 결과

| 항목 | 실제 확인 결과 |
|---|---|
| 원본 보존 | 원본 backend 폴더는 수정하지 않고 별도 복사본 준비 |
| Python 문법 | 파이썬 코드 4개의 AST 구문 분석 통과 |
| Compose | config 검증 통과; 실제 env 파일 값 해석은 제외한 구성 검사 |
| 자격 증명 | 업로드 파일에서 주요 키·암호 포함 URL 패턴 검사 통과 |
| 포함 파일 | 승인한 12개 파일과 일치 |
| 커밋 | 6fff6e5 |
| 메시지 | feat: AWS RDS 연결용 FastAPI 백엔드 배포 준비 |
| 브랜치 | main |
| GitHub push | 원격 main의 커밋 SHA 일치 확인 |
| 저장소 공개 범위 | PRIVATE 확인 |
| 작업 트리 | 커밋 이후 미반영 변경 없음 |

이 검사는 구문과 저장소 반영을 확인한 것이다. Docker 이미지를 실제로 빌드하거나 EC2에서 FastAPI를 시작한 결과는 아니다. private 저장소에서 EC2로 코드를 가져오려면 적절한 GitHub 인증 방법도 필요하다. 아직 인증을 EC2에 설정하지 않았다.

## 주제 6: 인바운드 규칙이 두 개인 이유 다시 확인

사용자가 “인바운드를 왜 두 개 만들었지?”라고 다시 질문했다. 이어 “EC2에서 지정해서 EC2 보안 그룹인 줄 알았다”고 말했다. 보안 그룹 관리 화면의 위치와 실제 규칙 적용 대상을 구분하는 것이 핵심이었다.

RDS 보안 그룹 하나 안에 다음 두 규칙을 둔다. 둘 다 PostgreSQL TCP 5432번 포트지만 허용하는 출발지가 다르다.

| RDS 인바운드 소스 | 허용하는 접속 | 이번에 확인한 결과 |
|---|---|---|
| 로컬 PC의 공인 IP 범위 | VS Code 접속, 추후 로컬 ingest.py 실행 | VS Code SQL 조회 성공 |
| EC2 보안 그룹 | EC2에서 RDS 접속, 추후 FastAPI의 DB 사용 | EC2 psql 로그인과 확장 조회 성공 |

보안 그룹 관리가 EC2 콘솔 메뉴 아래에 있어도 그 그룹이 RDS에 연결돼 있으면 RDS의 인바운드를 제어한다. 실습에서는 RDS용 그룹을 편집하면서 소스로 EC2용 그룹을 참조했다. 그룹 이름의 실제 학생 번호와 AWS 자원 ID는 기록하지 않는다.

> ♻️ 처음엔 EC2 메뉴에서 지정했으므로 EC2의 출입 규칙이라고 이해했으나, RDS에 연결된 보안 그룹을 편집하고 그 규칙의 소스로 EC2 보안 그룹을 선택한 것이 맞다. 뜻은 “이 EC2에서 오는 DB 접속을 RDS가 받아준다”이다.

> ➕ **더 알아두기 — 어디로 들어오는 규칙인가**
> 인바운드는 항상 해당 보안 그룹이 적용된 자원으로 들어오는 통신이다. 여기서는 목적지가 RDS이고 출발지가 로컬 PC 또는 EC2다. “메뉴는 EC2, 적용 대상은 RDS, 허용 소스는 EC2”가 동시에 성립할 수 있다. EC2의 SSH 22번 규칙과 RDS의 DB 5432번 규칙은 별개다.

> 📝 **기사 시험 포인트** (정보처리기사·정보보안기사)
> 접근 통제 규칙을 읽을 때 출발지(Source)·목적지(Destination)·프로토콜·포트를 함께 본다. 같은 포트라도 출발지가 다르면 허용 범위가 다르다. 관리 화면의 이름보다 정책이 적용되는 대상과 트래픽 방향을 기준으로 판단한다.

## 주제 7: 지금까지 완료한 범위와 다음 단계

교안의 백엔드 배포 순서는 저장소 준비 → Compose 작성 → EC2에서 코드 가져오기 → EC2 환경변수 설정 → 빌드·실행 → 상태 확인이다. 이번에는 배포용 저장소와 Compose 준비 및 GitHub 업로드까지 진행했다.

| 단계 | 상태·근거 |
|---|---|
| EC2에서 RDS 로그인 | TLSv1.3 연결과 aim_agent 프롬프트 캡처 확인 |
| pgvector 활성화 | CREATE EXTENSION 및 vector 0.8.2 조회 확인 |
| VS Code에서 RDS 접속 | Connection Success와 DB 트리 확인 |
| VS Code에서 실제 SQL | aim_agent / vector / 0.8.2 한 행 조회 확인 |
| 백엔드 코드 확인 | 사용자 지정 원본 폴더의 파일을 읽어 확인 |
| 배포용 Compose 준비 | 별도 복사본에 RDS용 구성 저장·설정 검사 |
| GitHub 저장소 생성 | private 생성·가시성 확인 |
| 커밋·push | 사용자 승인 후 실행, 원격 SHA 확인 |
| EC2 clone | 아직 실행하지 않음 |
| EC2의 실제 .env 작성 | 아직 실행하지 않음 |
| EC2 Docker build·up | 아직 실행하지 않음 |
| 배포 백엔드 /health 확인 | 아직 실행하지 않음 |
| ingest.py 데이터 적재·검색 | 아직 실행하지 않음 |
| Nginx·HTTPS·Vercel·CORS | 이후 단계, 아직 구성하지 않음 |

> ➕ **더 알아두기 — GitHub 업로드와 배포**
> push는 코드를 원격 저장소에 올리는 작업이다. EC2가 그 코드를 가져오고 실제 프로세스나 컨테이너를 실행해야 서비스가 동작한다. GitHub 업로드 성공만으로 배포가 끝났다고 판단하지 않는다. 지금은 DB 준비와 배포용 소스 전달 준비를 마친 상태다.

다음 수업에서는 private 저장소 인증 방법을 정하고 EC2에서 코드를 가져온다. 그 뒤 실제 .env를 작성하고, 인증서 경로를 확인하고, Docker Compose로 백엔드를 실행한다. 성공은 EC2의 컨테이너 상태·로그와 /health의 status=ok 응답으로 확인한다. 복사한 코드의 /health는 DB 연결 대상 정보도 응답하므로 교안의 짧은 예시와 JSON 전체가 완전히 같지 않을 수 있다.

데이터 적재는 뒤 단계이며 Gemini API 사용량이 발생한다. 이번 대화에서는 실행하지 않았다. TIL 요청을 이유로 환경변수 입력·외부 API 호출·EC2 작업을 추가로 수행하지 않는다.

## 정리

- RDS는 DB 서버이고 psql·VS Code는 그 DB에 접속하는 도구다. EC2와 로컬 PC 모두 같은 aim_agent에 접속했다.
- CA 인증서와 verify-full 설정을 사용해 EC2의 TLS 접속을 확인했다. 파일이 필요한 위치는 클라이언트가 실행되는 컴퓨터나 컨테이너다.
- pgvector 0.8.2를 활성화했고 VS Code에서도 같은 결과를 조회했다. 검색 문서 적재까지 완료된 것은 아니다.
- rdsadmin 거절은 AWS 내부 관리 DB에 관한 오류였고, aim_agent 조회는 성공했다. 접근 제한이나 SSL을 바꾸지 않았다.
- RDS 인바운드 두 줄은 PC와 EC2라는 두 출발지를 허용한다. EC2 메뉴에 있는 화면에서도 RDS 규칙을 편집할 수 있다.
- 로컬 DB용 YAML을 RDS용 배포 YAML로 따로 준비하고, 비밀 정보를 제외한 파일 12개를 승인 후 private 저장소에 push했다.
- 현재 단계는 GitHub 업로드 완료다. EC2에서 백엔드를 실행하고 /health를 확인하는 배포 작업은 다음 순서다.

## ✅ 확인 질문

1. postgres 이미지에서 psql을 실행하는 것과 PostgreSQL DB 서버를 새로 운영하는 것은 어떻게 다른가?
2. EC2 셸의 프롬프트와 psql의 프롬프트를 구분하고, CA 파일 마운트의 왼쪽·오른쪽 경로는 어디를 뜻하는지 설명하라.
3. CREATE EXTENSION IF NOT EXISTS vector와 pg_extension 조회는 각각 무엇을 하며 실제 확인한 버전은 무엇인가?
4. pgvector 활성화, 벡터 테이블 준비, ingest.py 데이터 적재, 검색은 왜 각각 다른 단계인가?
5. VS Code에서 RDS에 접속할 때 Host·Database·Username·Password·CA Certificate에는 각각 어떤 종류의 값을 넣는가?
6. EC2에 다운로드한 PEM 파일만으로 로컬 PC의 VS Code 설정까지 해결되지 않는 이유와 code .의 의미는 무엇인가?
7. 연결 성공 메시지와 rdsadmin 접근 거절이 함께 표시된 이유를 설명하고, 이 오류 때문에 보안 그룹이나 SSL을 바꿀 필요가 있었는지 말하라.
8. current_database()와 pg_extension을 조회한 결과는 어느 접속 대상과 어떤 기능을 검증했는가?
9. 로컬 DB가 포함된 Compose와 RDS에 접속하는 배포용 Compose는 어떻게 다르며, 127.0.0.1 바인딩은 어떤 접근 범위를 뜻하는가?
10. .env.example과 실제 .env, private 저장소와 비밀 정보 보호는 각각 어떻게 구분해야 하는가?
11. RDS 인바운드 규칙 두 줄은 각각 누구의 어떤 접속을 허용하며, EC2의 SSH 인바운드와는 어떻게 다른가?
12. 보안 그룹 화면이 EC2 메뉴에 있어도 RDS에 적용되는 규칙일 수 있는 이유를 목적지와 소스를 구분해 설명하라.
13. GitHub push 완료와 EC2 백엔드 배포 완료는 어떻게 다르며, 현재까지 실제로 검증한 작업은 어디까지인가?
14. EC2에서 코드를 가져온 다음 무엇을 설정하고 어떤 결과를 확인해야 백엔드 실행 성공이라고 판단할 수 있는가?

## 출처와 공개 기록 기준

- [강의 교안 — 배포](https://ucanlabs.kr/classroom/aicampus-agent-python/lesson/1478/) — 앞선 대화에서 읽은 RDS CA·psql·pgvector·백엔드 배포 절차를 기준으로 한다.
- 2026-11-02 사용자와의 학습 대화, 사용자 제공 psql·VS Code 화면 — 실제 실행과 확인 결과의 근거다. 이미지는 사용자 요청에 따라 첨부하지 않았다.
- 사용자가 지정한 backend의 실제 파일과 별도로 준비한 배포용 파일 — 코드 구성과 로컬·배포 설정 비교의 근거다.
- Git 커밋·push 출력 및 저장소 가시성 조회 — 파일 12개의 업로드와 PRIVATE 상태를 확인했다. private 저장소 주소는 공개 TIL에 별도로 링크하지 않는다.
- [AWS RDS PostgreSQL의 SSL 연결](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/PostgreSQL.Concepts.General.SSL.html) — TLS 연결·CA·인증서 검증 개념.
- [AWS RDS PostgreSQL 관리 활동](https://aws.amazon.com/blogs/database/working-with-rds-and-aurora-postgresql-logs-part-1/) — AWS 내부 관리 사용자와 관리 작업 설명.

교안 절차와 실제 실습은 일반 본문, 보충 설명은 ➕, 교정한 이해는 ♻️, 시험 개념 연결은 📝로 구분했다. 실제 암호·API 키·계정 ID·IP·엔드포인트·자원 ID·개인 절대 경로는 제외했다. 코드 예시의 127.0.0.1은 공통 루프백 주소이며 개인 서버 주소가 아니다.
