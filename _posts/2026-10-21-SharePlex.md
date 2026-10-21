---
title: "[SharePlex] Config 수정부터 Datapump 초기적재 및 Reconcile 복제 시나리오"
date: 2026-10-21 10:00:00 +0900
categories:
  - SharePlex
tags:
  - SharePlex
  - Oracle
  - Replication
  - Database
---

이번 포스팅에서는 SharePlex 환경에서 Config 파일을 수정하고, **Datapump(`expdp`/`impdp`)를 활용한 데이터 초기적재**, 그리고 **Reconcile과 Flush를 통한 큐 정합성 맞춤 및 검증 작업** 시나리오를 단계별로 살펴보겠습니다.

---

# 1. Config 수정 및 반영

기존 `old_config` 파일을 복사하여 `new_config` 파일을 생성하고, 대상 큐(Queue)를 중지한 뒤 `new_config`를 활성화합니다.

> 💡 **참고:** 여기서 **대상 큐(Queue)**는 초기적재 작업을 진행할 테이블이 속해있는 Post Queue를 의미합니다.

## 1.1. OS 환경에서 Config 복사 및 편집

```bash
# var/config 디렉토리 이동 후 설정 파일 복사
cp old_config new_config
ls -l
vi new_config
```

## 1.2. sp_ctrl 상에서 Post 중지 및 신규 Config 활성화

```text
# Target DB sp_ctrl
sp_ctrl (Target)> stop post queue <queue_name>

# Source DB sp_ctrl
sp_ctrl (Source)> activate config <new_config>
sp_ctrl (Source)> list config
```

## 1.3. 프로세스 상태 확인

Target DB에서 `show` 명령어를 수행하여 해당 Post 프로세스가 `Stopped by user` 상태인지 확인합니다.

```text
sp_ctrl (Target)> show
```

---

# 2. Datapump 초기적재

Datapump를 활용한 상세한 초기적재 작업 방법(Directory 생성, SCN 채번, `expdp`/`impdp` 옵션 등)은 아래 포스팅을 참고하시기 바랍니다.

👉 **[Datapump를 이용한 데이터 초기적재 가이드 바로가기]([URL_주소](https://kws9507-ops.github.io/kws-blog/oracle/2026/10/20/Oracle.html))**

---

# 3. Reconcile 수행 (Target)

Target DB에서 Datapump 시점 기준으로 Reconcile을 수행하여 초기적재 중 발생한 트랜잭션의 중복 적용을 방지합니다.

* **Datapump 사용 시:** `SCN_Number + 1` 값 사용
* **RMAN 백업/복구 사용 시:** `SCN_Number` 값 그대로 사용

> 💡 **운영환경 팁**  
> Reconcile 명령 실행 후 큐 대기 상태가 지속될 수 있습니다. 운영 환경에서는 임의의 DML을 실행할 수 없으므로 다음 단계의 `flush` 명령어로 대기 상태를 해제합니다.

```text
# Target DB sp_ctrl
sp_ctrl (Target)> reconcile queue <queue_name> for o.<source_sid>-o.<target_sid> scn <SCN_Number + 1>
```

---

# 4. Flush 및 Post 프로세스 구동

Source DB에서 큐를 플러시(Flush)하여 Reconcile 대기 상태를 풀고, Target DB의 Post 프로세스를 재시작합니다.

## 4.1. Source 및 Target 작업

```text
# 1) Source DB sp_ctrl에서 Flush 실행
sp_ctrl (Source)> flush o.<source_sid> queue <queue_name>

# 2) Target DB sp_ctrl에서 Post 시작
sp_ctrl (Target)> start post queue <queue_name>
```

## 4.2. 프로세스 상태 확인

Target DB에서 `show` 명령어로 `Post` 프로세스가 정상적으로 `Running` 상태로 전환되었는지 확인합니다.

```text
sp_ctrl (Target)> show
```

---

# 5. Compare 검증

모든 작업이 완료된 후 Source와 Target 데이터의 정합성을 검증합니다.

## 5.1. Compare 실행 옵션 예시

```text
# 신규 Config 기준 전체 비교
sp_ctrl (Source)> compare using <new_config>

# 병렬 처리 옵션을 활용한 비교 (대용량 테이블)
sp_ctrl (Source)> compare using <new_config> threads 2 parallelism 2 nosqllog

# 단일 테이블 또는 파티션 단위 비교
sp_ctrl (Source)> compare <schema>.<table_name>
sp_ctrl (Source)> compare <schema>.<table_name>.<partition_name>
```

## 5.2. 진행 상태 및 결과 확인

```text
# Compare 작업 상태 확인
sp_ctrl (Source)> compare status

# 상세 Job 결과 확인 (In Sync 여부 확인)
sp_ctrl (Source)> job status <job_id> full
```
