---
title: "[SharePlex] Object Cache 파일에서 테이블 목록 추출하기"
date: 2026-10-14 11:00:00 +0900
categories:
  - SharePlex
tags:
  - SharePlex
  - Oracle
  - Linux
  - Troubleshooting
  - ObjectCache
---

SharePlex 운영 및 장애 분석 과정에서 Object Cache 파일에 저장된 정보를 확인해야 하는 경우가 있다.

이번 글에서는 Linux 환경에서 SharePlex의 Object Cache 파일을 찾고, `dumpfile` 유틸리티를 이용해 파일 내용을 추출한 다음, 출력 결과에서 테이블명만 확인하는 방법을 정리한다.

## 1. 작업 개요

**작업 목적:** SharePlex Object Cache 파일의 내용을 덤프 파일로 출력하고, 출력 결과에서 테이블명을 확인한다.

작업 순서는 다음과 같다.

1. SharePlex 환경으로 이동한다.
2. Object Cache 파일의 위치와 최신 파일을 확인한다.
3. `dumpfile` 유틸리티를 실행한다.
4. Object Cache 파일을 지정하여 내용을 추출한다.
5. 추출한 결과에서 테이블명을 조회한다.

## 2. Object Cache 파일 확인

### 2-1. SharePlex 환경으로 이동

SharePlex 실행 환경에서 다음 명령어를 입력한다.

```bash
spvar
```

`spvar` 명령어를 통해 SharePlex의 변수 디렉터리로 이동하거나 해당 경로를 확인한다. 실제 동작은 환경에 따라 다를 수 있다.

이어서 `state` 디렉터리로 이동한다.

```bash
cd state
```

### 2-2. Object Cache 파일 목록 조회

다음 명령어로 Object Cache 관련 파일을 조회한다.

```bash
ls *objcache_sp_ocap*
```

파일명에 `objcache_sp_ocap` 문자열이 포함된 파일 목록을 확인할 수 있다.

예시:

```text
o.oracle-objcache_sp_ocap.21
o.oracle-objcache_sp_ocap.22
o.oracle-objcache_sp_ocap.23
```

### 2-3. 현재 경로 및 최신 파일 확인

현재 작업 디렉터리를 확인한다.

```bash
pwd
```

예시:

```text
/home/splexdr/splex/var/state
```

앞서 조회한 파일 목록을 기준으로 최신 파일을 확인한다. 파일명의 숫자가 가장 크다고 해서 반드시 가장 최근에 수정된 파일은 아니므로, 필요하면 수정 시간을 함께 확인한다.

```bash
ls -lt *objcache_sp_ocap*
```

예시:

```text
o.oracle-objcache_sp_ocap.23
```

**메모할 항목**

- Object Cache 파일의 전체 경로
- 확인한 파일명

예시 전체 경로:

```text
/home/splexdr/splex/var/state/o.oracle-objcache_sp_ocap.23
```

## 3. dumpfile 유틸리티 실행

### 3-1. SharePlex 바이너리 디렉터리로 이동

다음 명령어를 입력한다.

```bash
spbin
```

`spbin`은 SharePlex 바이너리 디렉터리로 이동하기 위해 사용하는 명령어 또는 셸 환경의 편의 명령어다. 환경에 따라 별도로 정의되어 있어야 한다.

### 3-2. dumpfile 실행

바이너리 디렉터리에서 다음 명령어를 실행한다.

```bash
./dumpfile
```

유틸리티의 메뉴가 표시되면 아래 순서대로 입력한다.

| 순서 | 입력 내용 |
|---|---|
| 1 | `4` 입력 |
| 2 | 확인한 Object Cache 파일의 전체 경로 입력 |
| 3 | `1` 입력 |
| 4 | 출력 파일의 전체 경로 입력 |

입력 경로 예시:

```text
/home/splexdr/splex/var/state/o.oracle-objcache_sp_ocap.23
```

출력 파일 경로 예시:

```text
/home/splexdr/bitekic.out
```

위 경로와 메뉴 번호는 작업 환경에서 사용하는 `dumpfile`의 실제 메뉴 구성에 맞춰 입력한다.

## 4. 출력 파일에서 테이블명 확인

`dumpfile` 실행이 완료되면 지정한 경로에 출력 파일이 생성된다.

다음 명령어로 출력 결과에서 큰따옴표(`"`)가 포함된 행을 조회한다.

```bash
cat /home/splexdr/bitekic.out | grep '"'
```

동일한 작업은 `grep`에 파일 경로를 직접 전달하여 수행할 수도 있다.

```bash
grep '"' /home/splexdr/bitekic.out
```

출력 결과 예시:

```text
"SCOTT"."EMP"
"SCOTT"."DEPT"
"HR"."EMPLOYEES"
```

위 내용은 출력 형식을 설명하기 위한 예시이며, 실제 테이블명은 덤프 파일의 내용에 따라 달라진다.

### 테이블명 조회 시 참고사항

큰따옴표가 포함된 행에 테이블명 외의 다른 정보가 있을 수도 있다. 따라서 이 명령어는 테이블명 후보를 빠르게 확인하기 위한 검색 방법이다.

실제 출력 형식에 따라 스키마명과 테이블명이 어떻게 표시되는지 확인한 후 필요한 정보를 정리한다.

## 5. 전체 작업 명령어 요약

Object Cache 파일 확인부터 덤프 파일 검색까지의 명령어를 순서대로 정리하면 다음과 같다.

```bash
# 1. SharePlex 변수 디렉터리로 이동
spvar

# 2. state 디렉터리로 이동
cd state

# 3. Object Cache 파일 목록 확인
ls *objcache_sp_ocap*

# 4. 현재 경로 확인
pwd

# 5. 파일 수정 시간 기준으로 목록 확인
ls -lt *objcache_sp_ocap*

# 6. SharePlex 바이너리 디렉터리로 이동
spbin

# 7. dumpfile 실행
./dumpfile
```

`dumpfile` 실행 후 메뉴에서 `4`를 입력하고 Object Cache 파일 경로를 지정한다. 이어서 `1`을 입력하고 출력 파일 경로를 지정한다.

출력 파일 생성 후에는 다음 명령어로 테이블명 후보를 확인한다.

```bash
grep '"' /home/splexdr/bitekic.out
```

## 6. 작업 시 주의사항

- **파일 경로 확인:** `pwd`로 현재 경로를 확인하고 Object Cache 파일의 전체 경로를 기록한다.
- **최신 파일 확인:** 파일명의 숫자만으로 판단하지 말고 `ls -lt`로 수정 시간을 함께 확인한다.
- **출력 파일 지정:** 기존 파일을 덮어쓰지 않도록 출력 파일명을 확인한다.
- **덤프 결과 확인:** `grep '"'` 검색 결과가 모두 테이블명이라는 보장은 없으므로 실제 출력 형식을 확인한다.
- **원본 파일 보호:** 분석 대상 Object Cache 파일은 임의로 수정하거나 삭제하지 않는다.

## 마무리

이번 글에서는 SharePlex의 Object Cache 파일을 확인하고, `dumpfile`을 사용해 내용을 출력한 다음, `grep` 명령어로 테이블명 후보를 조회하는 방법을 정리했다.

SharePlex 운영 중 Object Cache 내부 정보를 확인해야 할 때 이 절차를 참고하면 파일 확인부터 결과 검색까지의 작업을 일관된 순서로 진행할 수 있다.
