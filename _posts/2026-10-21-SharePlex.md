---
title: "[SharePlex] Queue분리 => 초기적재 => 복제 시나리오"
date: 2026-10-21 10:00:00 +0900
categories:
  - SharePlex
tags:
  - SharePlex
  - Oracle
  - Replication
  - Database
---

이번 포스팅에서는 SharePlex 환경에서 config파일을 수정하고, Datapump(`expdp`/`impdp`)를 활용한 데이터 동기화 과정, 그리고 Reconcile과 Flush를 통한 큐 정합성 맞춤 작업을 단계별로 살펴보겠습니다.

---

# 1. Config 수정 및 반영

기존 설정 파일을 복사하여 새로운 설정 파일을 생성하고, 대상 큐(Queue)를 중지한 뒤 새로운 설정을 활성화합니다.

```bash
# 설정 파일 복사 및 편집
cp old_config new_config
ls
vi new_config => Queue분리 작업
```

`sp_ctrl` 상에서 대상 큐를 중지하고 새로운 설정을 활성화합니다.

```text
sp_ctrl (oracle:2100)> stop post queue P08

sp_ctrl (rac1-vip.localdomain:2100)> activate config cfg_01_1

sp_ctrl (rac1-vip.localdomain:2100)> list config

File   Name                                         State       Datasource
--------------------------------------------------  ----------  ---------------
cfg_01                                              Inactive    o.SPLEX
Last Modified At: 06-Jan-26 15:24    Size: 727
ORA_config                                          Inactive    o.SOURCE_SID
Last Modified At: 05-Jan-26 16:35    Size: 151
cfg_01_1                                            Active      o.SPLEX
Last Modified At: 08-Jan-26 15:19    Size: 802     Internal Name: .conf.7
```

프로세스 상태를 확인하면 `P08` 프로세스가 사용자에 의해 중지(`Stopped by user`)된 것을 확인할 수 있습니다.

```text
sp_ctrl (oracle:2100)> show
Process    Source                               Target                 State                   PID
---------- ------------------------------------ ---------------------- -------------------- ------
Import     rac1-vip-E05                         oracle                 Running                3562
...
Post       o.SPLEX-P08                          o.oracle               Stopped by user
```

---

## 6. SourceDB에 데이터 추가

테스트를 위해 Source DB에 데이터를 100건 추가하고 커밋합니다.

```sql
INSERT INTO hr_sp_chat
	SELECT LEVEL + 100 AS room_id, 101 AS sender_id, 'SharePlex identical message' AS message, 1 AS send_seq, SYSDATE AS sent_date 
	FROM dual
	CONNECT BY LEVEL <= 100;
COMMIT;
```

---

## 7. 큐(Queue) 상태 확인

`qstatus detail` 명령어를 통해 각 Export, Capture, Post 큐의 상태와 메시지 적재 현황을 모니터링합니다.

```text
sp_ctrl (rac1-vip.localdomain:2100)> qstatus detail
sp_ctrl (oracle:2100)> qstatus detail
```

---

## 8. Datapump 준비

데이터베이스 내보내기/가져오기를 수행하기 위해 디렉토리 객체를 생성하고 권한을 부여합니다.

```sql
-- Source DB
SYS@racdb1> CREATE OR REPLACE DIRECTORY datapump AS '/u01/app/oracle/admin/racdb/dpdump';
Directory created.
SYS@racdb1> GRANT READ, WRITE ON DIRECTORY datapump TO splex;
Grant succeeded.

-- Target DB
SQL> CREATE OR REPLACE DIRECTORY datapump AS '/u01/app/oracle/admin/oracle/dpdump';
Directory created.
SQL> GRANT READ, WRITE ON DIRECTORY datapump TO splex;
Grant succeeded.
```

---

## 9. SCN 채번

동기화 기준점이 될 현재 SCN을 확인합니다.

```sql
SYS@racdb1> select current_scn from v$database;

CURRENT_SCN
-----------
    5601866
```

---

## 10. 데이터 Export 및 Import

`flashback_scn` 옵션을 사용하여 특정 시점의 데이터로 `expdp`를 수행한 뒤, Target DB에 `impdp`를 진행합니다.

```bash
# Export 수행
[racdb1:/home/splex]> expdp splex/splex directory=datapump tables=hr.HR_SP_CHAT dumpfile=hrdump.dmp logfile=SP.log flashback_scn=5601866

# Import 수행 (테이블이 존재할 경우 Truncate 옵션 사용)
[oracle:/home/splex]> impdp splex/splex directory=datapump dumpfile=hrdump.dmp logfile=SP.log table_exists_action=TRUNCATE
```

---

## 11. Reconcile 수행 (Target)

Datapump 사용 시 SCN 값에 +1을 더한 값으로 Reconcile을 수행합니다.

> **💡 운영환경 팁**  
> 무한 대기 상태를 풀기 위해 임의의 `insert`를 실행할 수 없는 운영환경에서는 `flush` 명령어로 해결합니다.

```text
sp_ctrl (oracle:2100)> reconcile queue P08 for o.SPLEX-o.oracle scn 5355447 
```

---

## 12. Flush 및 Post 프로세스 구동

큐를 플러시(Flush)한 후 `P08` 포스트 프로세스를 다시 시작합니다.

```text
sp_ctrl (rac1-vip.localdomain:2100)> flush o.SPLEX queue P08
sp_ctrl (oracle:2100)> start post queue P08
```

프로세스 상태를 조회하여 `P08`이 정상적으로 `Running` 상태로 전환되었는지 확인합니다.

```text
sp_ctrl (oracle:2100)> show
Process    Source                               Target                 State                   PID
---------- ------------------------------------ ---------------------- -------------------- ------
...
Post       o.SPLEX-P08                          o.oracle               Running                4865
```

---

## 13. Compare 검증

설정이 완료된 후 데이터 정합성을 확인하기 위해 `compare`를 수행합니다.

```text
sp_ctrl (rac1-vip.localdomain:2100)> compare using cfg_01_1
  comparing 8 of 8 objects
  compare started; job id 2
```

작업 상태를 모니터링합니다.

```text
sp_ctrl (rac1-vip.localdomain:2100)> compare status
sp_ctrl (rac1-vip.localdomain:2100)> job status 2 full
```

### 🔍 검증 결과

모든 테이블이 `In Sync` 상태로 정상적으로 동기화된 것을 확인할 수 있습니다.

```text
ID     Tablename                            Status         Time         Total Rows %Comp Total Time
------ ------------------------------------ -------------- ------------ ---------- ----- ------------
1      "HR"."REGIONS"                       In Sync                 N/A          4   100         0:08
2      "HR"."COUNTRIES"                     In Sync                 N/A         26   100         0:07
3      "HR"."LOCATIONS"                     In Sync                 N/A         23   100         0:06
4      "HR"."DEPARTMENTS"                   In Sync                 N/A         27   100         0:06
5      "HR"."JOBS"                          In Sync                 N/A         19   100         0:06
6      "HR"."EMPLOYEES"                     In Sync                 N/A        107   100         0:05
7      "HR"."JOB_HISTORY"                   In Sync                 N/A         10   100         0:06
8      "HR"."HR_SP_CHAT"                    In Sync                 N/A        200   100         0:07
```
