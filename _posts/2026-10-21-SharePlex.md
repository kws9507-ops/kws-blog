---
title: "[SharePlex] Queue수정 => 초기적재 => 복제 시나리오"
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

old_config파일을 복사하여 new_config파일을 생성하고, 대상 큐(Queue)를 중지한 뒤 new_config를 활성화합니다.
여기서 대상 큐(Queue)는 초기적재 작업을 진행한 테이블이 속해있는 Queue.

```bash
# old_config파일 복사 및 편집
cp old_config new_config
ls
vi new_config
```

`sp_ctrl` 상에서 대상 큐(Queue)를 중지하고 새로운 설정을 활성화합니다.
여기서 대상 큐(Queue)는 초기적재 작업을 진행한 테이블이 속해있는 Queue.

```text
stop post queue queue_name
activate config new_config
list config
```

프로세스 상태를 확인하면 `대상 큐(Queue)` 프로세스가 사용자에 의해 중지(`Stopped by user`)된 것을 확인할 수 있습니다.

```text
show
```

---

# 2. Datapump 초기적재


---

# 3. Reconcile 수행 (Target)

Datapump 사용 시 SCN 값에 +1을 더한 값으로 Reconcile을 수행합니다.
RMAN 사용 시 SCN 값 그대로 사용

> **💡 운영환경 팁**  
> 무한 대기 상태를 풀기 위해 임의의 `insert`를 실행할 수 없는 운영환경에서는 `flush` 명령어로 해결합니다.

```text
reconcile queue 대상 큐(Queue) for o.source-o.target scn SCN_Number+1 
```

---

# 4. Flush 및 Post 프로세스 구동

큐를 플러시(Flush)한 후 `대상 큐(Queue)` 포스트 프로세스를 다시 시작합니다.

```text
<Source Server>
flush o.source queue 대상 큐(Queue)
<Target Server>
start post queue 대상 큐(Queue)
```

프로세스 상태를 조회하여 `대상 큐(Queue)`이 정상적으로 `Running` 상태로 전환되었는지 확인합니다.

```text
show
```

---

# 5. Compare 검증

설정이 완료된 후 데이터 정합성을 확인하기 위해 `compare`를 수행합니다.

```text
compare using cfg_01_1

```

작업 상태를 모니터링합니다.

```text
compare status
job status job_id full
```
