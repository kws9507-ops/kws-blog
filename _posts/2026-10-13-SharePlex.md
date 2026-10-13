---
title: "[SharePlex] SharePlex 설치 및 기본 설정 가이드"
date: 2026-10-13 10:00:00 +0900
categories:
  - SharePlex
tags:
  - SharePlex
  - Oracle
  - Replication
  - Database
---

Oracle Database 환경에서 데이터 실시간 복제(Replication)를 구현하기 위한 **Quest SharePlex** 설치 및 초기 구성 가이드입니다. OS 설정부터 DB 설정, 그리고 SharePlex 본 설치까지의 전 과정을 다루며, 각 작업이 수행되는 위치(**Source** 또는 **Target**)를 함께 표기하였습니다.

---

# 1. OS 설정

SharePlex를 구동하기 위한 운영체제 레벨의 사전 준비 작업입니다. (소스 및 타겟 서버 양쪽 모두 수행 필요)

## 1.1 SharePlex 계정 생성 및 관리 **[Source / Target 공통]**
SharePlex 데몬 및 프로세스를 실행할 전용 OS 계정을 생성합니다. 보안 및 권한 분리를 위해 일반 사용자 계정과 분리하는 것이 좋습니다.

```bash
# splex 계정 생성 예시
useradd splex
passwd splex
id oracle 확인
usermod -g oinstall -G dba splex
id splex 확인
```

### 1.2 .bash_profile 수정 **[Source / Target 공통]**
SharePlex 실행에 필요한 환경 변수(`SP_SYS_KEY`, `PATH` 등) 및 Oracle 환경 변수를 설정합니다.

```bash
# /home/splex/.bash_profile에 추가
export 오라클 계정과 동일
export SP_SYS_PRODDIR=/home/splex/splex/prod
export SP_SYS_VARDIR=/home/splex/splex/var
export SP_SYS_BINDIR=/home/splex/splex/prod/bin
export SP_SYS_HOST_NAME=rac1-vip
export SP_COP_TPORT=2100
export SP_COP_UPORT=2100
alias spbin='cd /home/splex/splex/prod/bin'
alias spvar='cd /home/splex/splex/var'
alias splog='cd /home/splex/splex/var/log'
alias sp='/home/splex/splex/prod/bin/sp_ctrl'

. .bash_profile
```

### 1.3 /etc/hosts 설정 **[Source / Target 공통]**
클러스터 환경 또는 네트워크 통신을 위해 호스트 이름과 IP 주소를 정확히 매핑합니다.

```text
# /etc/hosts 예시
192.168.1.100   db-source
192.168.1.101   db-target
```

### 1.4 /etc/oratab 설정 **[Source / Target 공통]**
인스턴스 자동 기동 및 Oracle 홈 디렉터리 인식을 위해 `/etc/oratab` 파일을 확인하고 설정합니다.

```text
# /etc/oratab 예시
ORCL:/u01/app/oracle/product/19.0.0/dbhome_1:Y
```

### 1.5 SharePlex OS User의 ulimit 설정 **[Source / Target 공통]**
SharePlex가 원활히 동작할 수 있도록 OS 리소스 제한 설정을 변경합니다.

```ini
# /etc/security/limits.conf 설정 예시
splex   soft    nofile    65536
splex   hard    nofile    65536
splex   soft    nproc     16384
splex   hard    nproc     16384
```

---

## 2. DB 설정

SharePlex가 데이터베이스와 연동하여 동작할 수 있도록 데이터베이스 내부 설정을 구성합니다.

### 2.1 TNS 설정 **[Source / Target 공통]**
소스와 타겟 간 원활한 접속(양방향 또는 네트워킹)을 위해 `tnsnames.ora` 파일을 구성합니다.

```text
ORCL =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = db-source)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = ORCL)
    )
  )
```

### 2.2 SharePlex TableSpace 생성 **[Source / Target 공통]**
SharePlex 메타데이터 및 큐(Queue) 관리를 위한 전용 테이블스페이스를 생성합니다.

```sql
CREATE TABLESPACE splex_ts 
DATAFILE '/u01/app/oracle/oradata/ORCL/splex_ts01.dbf' SIZE 2G 
AUTOEXTEND ON NEXT 500M MAXSIZE UNLIMITED;
```

### 2.3 SharePlex Oracle 계정 생성 및 권한 부여 **[Source / Target 공통]**
복제 작업을 수행할 전용 데이터베이스 사용자를 생성하고 필요한 권한을 부여합니다.

```sql
CREATE USER splex IDENTIFIED BY password 
DEFAULT TABLESPACE splex_ts 
TEMPORARY TABLESPACE temp;

GRANT CONNECT, RESOURCE TO splex;
GRANT SELECT ANY DICTIONARY TO splex;
-- 추가적인 필수 권한 부여 (버전에 따라 상이할 수 있음)
```

### 2.4 Parameter 확인 및 설정 **[Source / Target 공통]**
Oracle 아카이브 로그 모드 및 SharePlex 구동에 필요한 데이터베이스 파라미터를 점검합니다.

```sql
-- 아카이브 로그 모드 확인
ARCHIVE LOG LIST;

-- 필요 시 아카이브 모드 전환 (MOUNT 단계에서 수행)
-- ALTER DATABASE ARCHIVELOG;
```

### 2.5 Supplemental logging 설정 **[Source 전용]**
> ⚠️ **참고:** Supplemental Logging은 기본적으로 변경 데이터를 캡처해야 하는 **Source DB** 측에 설정합니다. (단, 타겟 측에서 향후 리버스(Reverse) 복제나 충돌 해결 등을 고려한다면 타겟에도 설정할 수 있습니다.)

```sql
-- 최소 Supplemental Logging 설정
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;

-- 특정 테이블에 대한 Supplemental Logging 설정 예시
ALTER TABLE schema.table ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
```

---

## 3. SharePlex 설치

실제 SharePlex 바이너리를 설치하고 초기 구성을 완료하는 단계입니다.

### 3.1 설치 경로 생성 **[Source / Target 공통]**
SharePlex 프로그램 디렉터리와 가변 데이터 디렉터리(`vardir`)를 생성합니다.

```bash
mkdir -p /splex/proddir
mkdir -p /splex/vardir
chown -R splex:splex /splex
```

### 3.2 SharePlex 설치 **[Source / Target 공통]**
다운로드한 SharePlex 설치 파일을 실행하여 설치를 진행합니다.

```bash
# 설치 파일 실행 예시
./SharePlex_Installer.bin
```
* 설치 마법사 지시에 따라 제품 디렉터리 및 가변 디렉터리 경로를 지정합니다.

### 3.3 ora.setup **[Source / Target 공통]**
설치 완료 후 Oracle 데이터베이스와 SharePlex를 연동하기 위한 설정 스크립트를 실행합니다.

```bash
# vardir 디렉터리 이동 후 설정 스크립트 실행
cd /splex/vardir
./ora.setup
```
* 프롬프트에 따라 Oracle Home 경로, TNS 이름, SharePlex 전용 DB 계정 정보를 입력하여 구성을 완료합니다.

---

> **참고:** 설치 완료 후 `sp_cop &` 명령어를 통해 각 서버에서 SharePlex 프로세스를 백그라운드로 실행하고, `sp_ctrl` 유틸리티를 통해 정상 구동 여부를 확인할 수 있습니다.
