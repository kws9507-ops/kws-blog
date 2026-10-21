---
title: "[SharePlex] SharePlex Source 서버 복제 제외 테이블 설정 방법 (SP_OPO_DISABLE_OBJECT_NUM)"
date: 2026-10-20 10:00:00 +0900
categories:
  - SharePlex
tags:
  - SharePlex
  - Oracle
  - Replication
  - Database
---

Oracle 데이터 가드(Data Guard)나 SharePlex 같은 CDC/복제 솔루션을 사용하여 DR(Disaster Recovery) 환경을 구축할 때, 특정 테이블(예: 대용량 로그 테이블, 임시 작업 테이블 등)은 복제 대상에서 제외해야 하는 경우가 있습니다. 

이번 글에서는 SharePlex 환경에서 특정 테이블을 복제 대상에서 제외하는 파라미터 설정 방법을 정리해 둡니다.

---

# 1. 핵심 요약: Source에도 적용해야 하는가?

**아닙니다. Source 서버에는 적용할 필요가 없으며, Target 서버에만 적용합니다.**

* **이유:** `SP_OPO_DISABLE_OBJECT_NUM` 파라미터 이름의 **OPO**는 **Operation Post**를 의미합니다. Post 프로세스는 Target 서버에서 큐(Queue)에 들어온 트랜잭션을 받아 Target DB에 반영(Apply/Post)하는 역할을 담당합니다.
* **그렇다면 왜 Source DB의 OBJECT_ID를 사용하는가?:** SharePlex는 캡처(Capture) 단계부터 **Source DB 기준의 Object ID**를 트랜잭션 헤더에 포함시켜 전달합니다. 따라서 Target Post 프로세스가 "어떤 테이블의 데이터인가"를 판단할 때 기준이 되는 값은 **Source DB의 OBJECT_ID**입니다.
* **결론:** **Source DB**에서 `OBJECT_ID`를 조회한 후, 해당 ID 값을 **Target 서버의 SharePlex(sp_ctrl)** 파라미터에 설정하면 됩니다.

---

# 2. 복제 제외 테이블 설정 절차

### 2.1 Source DB에서 제외할 테이블의 OBJECT_ID 확인
Source DB에 접속하여 복제에서 제외할 대상 테이블의 `OBJECT_ID`를 조회합니다.

```sql
SELECT owner, object_name, object_id, object_type
FROM dba_objects
WHERE owner = 'HR' AND object_name = '제외할테이블명';
```
*(예시: 조회된 `OBJECT_ID`가 `72965`라고 가정)*

## 2.2 Target 서버의 SharePlex 환경에서 파라미터 설정
Target 서버로 이동하여 SharePlex 제어 콘솔(`sp_ctrl`)에 접속한 뒤 파라미터를 변경합니다.

1. `sp_ctrl` 접속 후 현재 파라미터 확인
   ```bash
   sp_ctrl
   ```
   ```text
   sp_ctrl> list param all
   ```
   *`SP_OPO_DISABLE_OBJECT_NUM` 값을 검색하여 현재 설정값을 확인합니다.*

2. 파라미터 값 설정
   ```text
   sp_ctrl> set param SP_OPO_DISABLE_OBJECT_NUM 72965
   ```
   * **기존 설정값이 존재하는 경우:** 기존 값을 덮어쓰지 않도록 기존 `OBJECT_ID` 목록에 콤마(`,`)로 구분하여 함께 등록해야 합니다. (예: 기존값이 `12345`라면 `set param SP_OPO_DISABLE_OBJECT_NUM 12345,72965`)

---

# 3. 주의사항 및 체크리스트

1. **Post 프로세스 적용 확인:** 파라미터를 변경한 후 Post 프로세스에 즉시 반영되는지 확인하고, 필요한 경우 Post 프로세스를 재시작(`stop post` -> `start post`)합니다.
2. **Target DB 데이터 정리:** 파라미터를 설정하면 설정 이후 들어오는 해당 테이블의 변경사항만 Post 과정에서 무시(Ignore)됩니다. 복제 제외 처리 이전에 이미 Target DB에 적용되어 있던 데이터는 자동으로 삭제되지 않으므로 필요 시 수동으로 정리해야 합니다.
