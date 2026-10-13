---
title: "[Database] SharePlex - 오라클 환경 SharePlex 설치 및 기본 설정 가이드"
date: 2026-10-08 11:00:00 +0900
categories:
  - Database
  - SharePlex
tags:
  - SharePlex
  - Oracle
  - Replication
  - Database
---

오라클(Oracle) 환경에서 데이터베이스 실시간 복제(Replication)를 위해 퀘스트(Quest)사의 **SharePlex**를 설치하고 기본 환경을 설정하는 과정을 정리한다.

이번 글에서는 소스(Source) 및 타겟(Target) 서버에 SharePlex를 설치하고, 라이선스 등록 및 포트 설정, 그리고 데몬 구동까지 직접 실습했다.

---

# 1. SharePlex란?

**SharePlex는 오라클 데이터베이스를 위한 실시간 데이터 복제 및 고가용성(HA) 솔루션**이다.

기존 데이터베이스의 로그(Redo Log / Archivelog)를 분석하여 타겟 데이터베이스에 실시간으로 반영하기 때문에 성능 저하를 최소화하면서 데이터를 동기화할 수 있다.

예를 들어 시스템 구성은 다음과 같이 이루어진다.

```text
Source DB (Oracle)
  │
  │ Redo Log 분석
  ▼
SharePlex Capture
  │
  │ Queue 전송
  ▼
SharePlex Post
  │
  │ SQL 반영
  ▼
Target DB (Oracle)
```

운영(Source) DB에 부하를 주지 않고 데이터 실시간 백업 및 이중화를 구축할 수 있다는 장점이 있다.

---

# 2. 설치 사전 준비 사항

SharePlex를 설치하기 전에 다음 사항들을 확인해야 한다.

- 오라클 데이터베이스가 정상적으로 설치 및 구동 중인지 확인
- SharePlex 전용 OS 계정(예: `splex`) 생성
- 필요한 환경 변수(`ORACLE_HOME`, `ORACLE_SID` 등) 설정
- 설치 파일 확보 및 권한 설정

---

# 3. SharePlex 전용 OS 계정 생성

SharePlex는 보안과 관리를 위해 별도의 OS 계정으로 실행하는 것이 권장된다.

root 계정에서 다음 명령어로 계정을 생성했다.

```bash
useradd splex
passwd splex
```

이후 SharePlex 설치 및 실행은 `splex` 계정으로 진행한다.

---

# 4. SharePlex 설치 파일 압축 해제

`splex` 계정으로 로그인한 뒤, 다운로드받은 SharePlex 설치 파일을 작업 디렉토리에 위치시키고 압축을 해제했다.

```bash
tar -xvf SharePlex_Installer.tar
```

설치 파일 실행을 준비하는 과정이다.

---

# 5. SharePlex 설치 실행

압축 해제된 디렉토리에서 설치 스크립트를 실행한다.

```bash
./splex_install.sh
```

설치 과정에서 입력해야 하는 주요 항목은 다음과 같다.

```text
1. 설치 경로 (Product Directory)
2. 변수 경로 (Variable-data Directory)
3. 오라클 버전 선택 (예: Oracle 19c)
4. 포트 번호 설정 (Port Number)
```

설치가 완료되면 지정한 경로에 SharePlex 바이너리와 환경 설정 파일들이 생성된다.

---

# 6. 환경 변수 설정

`splex` 계정의 `.bash_profile`에 SharePlex 구동을 위한 환경 변수를 추가했다.

```bash
export SP_SYS_KEY=<System Key>
export SP_COP_CL_PORT=2100
export PATH=$PATH:/splex/product/bin
```

적용을 위해 다음 명령어를 실행했다.

```bash
source ~/.bash_profile
```

---

# 7. 라이선스 등록

SharePlex를 구동하기 전 라이선스 키를 등록해야 한다.

`splex` 계정으로 SharePlex 관리 툴에 진입한다.

```bash
sp_cop -u2100 &
```

이후 `ora_setup` 또는 라이선스 등록 명령어를 사용하여 발급받은 라이선스 키를 입력한다.

```text
SharePlex License Registration
  ↓
라이선스 키 입력
  ↓
등록 완료
```

---

# 8. SharePlex 데몬 구동 (sp_cop)

SharePlex의 핵심 프로세스인 `sp_cop`를 백그라운드 데몬으로 실행한다.

```bash
sp_cop -u2100 &
```

실행 상태 확인:

```bash
ps -ef | grep sp_cop
```

결과:

```text
splex   12345     1  0 10:00 ?        00:00:02 ./sp_cop -u2100
```

SharePlex가 정상적으로 구동된 것을 확인했다.

---

# 9. SharePlex 관리 콘솔 접속

SharePlex 상태를 모니터링하고 설정하기 위해 관리 콘솔(sp_ctrl)에 접속했다.

```bash
sp_ctrl
```

접속 결과 프롬프트가 변경된다.

```text
shareplex>
```

여기서 각종 SharePlex 명령어(status, list 등)를 사용할 수 있다.

---

# 10. 상태 확인 및 기본 명령어

`sp_ctrl` 콘솔 내에서 다음 명령어를 실행하여 상태를 점검했다.

```text
shareplex> show
```

실행 결과, 현재 구동 중인 프로세스들과 포트 상태가 정상적으로 표시되는 것을 확인했다.

---

# 11. 지금까지 배운 SharePlex 명령어

| 명령어 / 작업 | 설명 |
| :--- | :--- |
| `useradd splex` | SharePlex 전용 OS 계정 생성 |
| `./splex_install.sh` | SharePlex 설치 스크립트 실행 |
| `sp_cop -u[포트]` | SharePlex 메인 데몬 구동 |
| `ps -ef | grep sp_cop` | 데몬 프로세스 구동 확인 |
| `sp_ctrl` | SharePlex 관리 콘솔 진입 |
| `show` | SharePlex 프로세스 및 상태 확인 |

---

# 12. 전체 SharePlex 설치 및 구동 흐름

이번 실습에서 진행한 전체적인 구조는 다음과 같다.

```text
          OS 계정 생성 (splex)
                   │
                   ▼
           설치 파일 압축 해제
                   │
                   ▼
           splex_install.sh 실행
                   │
                   ▼
           환경 변수(.bash_profile) 설정
                   │
                   ▼
           라이선스 키 등록
                   │
                   ▼
            sp_cop 데몬 구동
                   │
                   ▼
           sp_ctrl 콘솔 접속 및 확인
```

앞으로 복제 대상 테이블 설정(Datasource 설정 및 `q_add` 등)을 진행할 때 이 기본 구동 흐름을 바탕으로 작업하게 된다.

---

# 13. SharePlex를 사용하는 이유

SharePlex는 오라클 데이터베이스 간의 대용량 데이터 복제를 실시간으로 처리할 수 있다.

기존 스토리지 기반 복제나 덤프 백업 방식과 달리, 트랜잭션 단위로 로그를 분석해 반영하기 때문에 데이터 정합성을 유지하면서 부하를 최소화할 수 있다.

예를 들어 장애 발생 시:

```text
Source DB 장애 발생
       │
       ▼
Target DB로 전환 (Failover)
       │
       ▼
서비스 중단 시간 최소화
```

와 같은 방식으로 고가용성 시스템을 운영할 수 있다.

---

# 마무리

이번 실습에서는 오라클 환경에서 SharePlex를 도입하기 위한 기초 과정부터

- OS 전용 계정 생성
- 설치 스크립트 실행
- 환경 변수 설정
- 라이선스 등록
- `sp_cop` 데몬 구동
- `sp_ctrl` 관리 콘솔 활용

까지 직접 진행했다.

설치와 기본 구동이 정상적으로 완료되었으므로, 다음 단계에서는 실질적인 Replication 설정 및 큐(Queue) 구성, 테이블 파라미터 설정을 진행할 예정이다.