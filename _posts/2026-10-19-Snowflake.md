---
title: "[Snowflake] Snowflake CoWork 에이전트를 위한 t검정 커스텀 도구 만들기"
date: 2026-10-19 10:00:00 +0900
categories:
  - Snowflake
tags:
  - Snowflake
  - t_test
---

> Cortex Analyst(시맨틱 뷰)만으로는 어려운 **t검정과 p-value 계산**을 Python 저장 프로시저로 구현해, Snowflake CoWork 에이전트의 커스텀 도구로 쓸 수 있게 만드는 과정을 정리했습니다.

# 목차
1. [왜 커스텀 도구가 필요한가](#1-왜-커스텀-도구가-필요한가)
2. [전체 구성](#2-전체-구성)
3. [샘플 데이터 만들기](#3-샘플-데이터-만들기)
4. [t검정 프로시저 만들기](#4-t검정-프로시저-만들기)
5. [실행 및 결과 해석](#5-실행-및-결과-해석)
6. [에이전트에 연결할 때 팁](#6-에이전트에-연결할-때-팁)
7. [마무리](#7-마무리)

---

# 1. 왜 커스텀 도구가 필요한가

Snowflake CoWork(구 Snowflake Intelligence)의 Cortex Analyst 도구는 자연어 질문을 **SQL로 바꿔 실행**합니다. 그래서 평균, 표준편차, 빈도, 상관계수 같은 기술통계는 시맨틱 뷰만으로 충분히 처리됩니다.

t검정은 상황이 다릅니다.

| 항목 | SQL만으로 처리할 때 |
|---|---|
| t값 계산 | `AVG`, `STDDEV_SAMP`, `COUNT`, `SQRT`로 가능 |
| 판정 기준(임계값) | 자유도에 따라 달라져서 고정값을 쓸 수 없음 |
| p-value | t분포의 누적분포함수가 Snowflake SQL에 기본 제공되지 않음* |
| Welch 자유도 | 공식이 복잡해서 LLM이 매번 정확하게 생성하기 어려움 |
| 전제조건 검정 | 정규성(Shapiro), 등분산(Levene) 검정은 SQL로 구현하기 어려움 |

\* 작성 시점 기준으로 확인한 내용입니다. 이후 Snowflake에 관련 함수가 추가되었을 수 있으니 공식 문서를 함께 확인하세요.

그래서 `scipy.stats`를 쓰는 **Python 저장 프로시저**를 만들고, 이를 에이전트의 custom tool로 등록하는 방식을 택했습니다.

# 2. 전체 구성

```
KWS_DB
└── STATISTICS (schema)
    ├── SAMPLE_TTEST     -- 테스트용 샘플 테이블
    └── RUN_TTEST(...)   -- t검정 Python 저장 프로시저 (VARIANT 반환)
```

동작 흐름은 다음과 같습니다.

```
사용자 질문 (CoWork)
  → 에이전트가 RUN_TTEST 도구를 선택하고 파라미터를 채움
  → 프로시저가 데이터를 읽어 scipy로 검정
  → JSON 결과 반환
  → 에이전트가 결과를 해석해서 답변
```

스키마는 다음과 같이 만듭니다.

```sql
CREATE SCHEMA IF NOT EXISTS KWS_DB.STATISTICS;
```

# 3. 샘플 데이터 만들기

## 3.1 시나리오

온라인 쇼핑몰에서 **새로운 프로모션(B안)** 을 테스트하는 상황을 가정했습니다.

- 고객 200명을 무작위로 **A그룹(기존안)** 과 **B그룹(신규 프로모션)** 으로 나눕니다.
- 각 고객의 **프로모션 전(BEFORE)** 과 **프로모션 후(AFTER)** 구매금액을 기록합니다.

이렇게 구성하면 하나의 테이블로 t검정 세 가지를 모두 테스트할 수 있습니다.

| 검정 유형 | 이 데이터에서 묻는 질문 |
|---|---|
| 일표본 t검정 | 프로모션 전 평균 구매금액이 50,000원과 다른가? |
| 독립표본 t검정 | 프로모션 후 A그룹과 B그룹의 구매금액이 다른가? |
| 대응표본 t검정 | 같은 고객의 구매금액이 프로모션 전후로 달라졌는가? |

## 3.2 테이블 구조

| 컬럼 | 타입 | 설명 |
|---|---|---|
| `CUSTOMER_ID` | NUMBER | 고객 ID (1~200). 대응표본 검정에서 쌍을 식별하는 키 |
| `AB_GROUP` | VARCHAR | 실험 그룹. `A`(기존안) 또는 `B`(신규 프로모션), 약 50:50 무작위 배정 |
| `PERIOD` | VARCHAR | 측정 시점. `BEFORE` 또는 `AFTER` |
| `PURCHASE_AMT` | NUMBER | 구매금액(원) |

고객 1명당 BEFORE와 AFTER 2행이 있으므로 **전체 400행**입니다.

## 3.3 데이터 생성 규칙

| 구간 | 분포 |
|---|---|
| BEFORE (전체 고객) | 정규분포, 평균 50,000 / 표준편차 10,000 |
| AFTER, A그룹 | 정규분포, 평균 51,000 / 표준편차 10,000 |
| AFTER, B그룹 | 정규분포, 평균 51,000 / 표준편차 10,000 **+ 4,000** |

즉 **B그룹 프로모션 효과가 약 4,000원**이 되도록 의도적으로 설계했습니다. 검정 도구가 이 차이를 제대로 잡아내는지 확인하는 것이 목적입니다.

## 3.4 생성 SQL

```sql
CREATE OR REPLACE TABLE KWS_DB.STATISTICS.SAMPLE_TTEST AS
WITH c AS (
  SELECT SEQ4() + 1 AS CUSTOMER_ID,
         IFF(UNIFORM(0, 1, RANDOM()) = 0, 'A', 'B') AS AB_GROUP
  FROM TABLE(GENERATOR(ROWCOUNT => 200))
)
SELECT CUSTOMER_ID, AB_GROUP, 'BEFORE' AS PERIOD,
       ROUND(NORMAL(50000, 10000, RANDOM()), 0) AS PURCHASE_AMT
FROM c
UNION ALL
SELECT CUSTOMER_ID, AB_GROUP, 'AFTER' AS PERIOD,
       ROUND(NORMAL(51000, 10000, RANDOM()), 0) + IFF(AB_GROUP = 'B', 4000, 0)
FROM c;
```

> **참고:** 시드 없이 `RANDOM()`을 쓰기 때문에 실행할 때마다 값이 달라집니다. 아래 결과 수치는 필자의 실행 결과이며, 직접 실행하면 숫자는 조금씩 다르게 나옵니다. 다만 설계한 효과(약 4,000원)가 있으므로 결론은 대체로 비슷하게 나옵니다.

## 3.5 생성된 데이터 요약 (필자 실행 기준)

| 구분 | n | 평균 | 표준편차 |
|---|---|---|---|
| BEFORE 전체 | 200 | 49,023 | 9,989 |
| AFTER 전체 | 200 | 54,023 | 10,328 |
| AFTER, A그룹 | 100 | 51,323 | 11,128 |
| AFTER, B그룹 | 100 | 56,723 | 8,709 |

# 4. t검정 프로시저 만들기

## 4.1 설계 포인트

- **세 가지 검정을 하나의 도구로:** `TEST_TYPE` 파라미터로 `one_sample`, `independent`, `paired`를 선택합니다. 에이전트가 고를 도구가 하나뿐이라 호출이 단순해집니다.
- **이름 기반 기본값 파라미터:** 필요한 인자만 넘기면 되므로 에이전트가 호출하기 쉽습니다.
- **입력 검증:** 테이블명과 컬럼명을 정규식으로 검증하고, 실제 테이블에 있는 컬럼인지 확인합니다. 쿼리는 문자열을 이어 붙이지 않고 Snowpark DataFrame API로 구성해서 SQL 인젝션을 막습니다.
- **해석하기 쉬운 결과:** 통계량과 함께 전제조건 검정, 효과크기, 판정 문구를 JSON으로 반환해서 LLM이 바로 해석할 수 있게 합니다.
- **오류도 JSON으로 반환:** 예외가 나도 `{"error": "..."}` 형태로 돌려주기 때문에, 에이전트가 사용자에게 원인을 설명할 수 있습니다.

## 4.2 파라미터

| 파라미터 | 기본값 | 설명 |
|---|---|---|
| `TABLE_NAME` | (필수) | 분석할 테이블. `DB.SCHEMA.TABLE` 형식 |
| `VALUE_COL` | (필수) | 검정할 수치 컬럼 |
| `TEST_TYPE` | `independent` | `one_sample` / `independent` / `paired` |
| `GROUP_COL` | NULL | 그룹 컬럼. 대응표본에서는 조건 컬럼(예: `PERIOD`) |
| `GROUP_A`, `GROUP_B` | NULL | 비교할 두 그룹 값. 평균 차이는 A − B로 계산 |
| `PAIR_ID_COL` | NULL | 대응표본에서 같은 대상을 식별하는 키 |
| `MU0` | 0 | 일표본 검정의 기준값 |
| `ALPHA` | 0.05 | 유의수준 |
| `FILTER_COL`, `FILTER_VALUE` | NULL | 선택 사항. `FILTER_COL = FILTER_VALUE` 조건으로 데이터를 좁힘 |

## 4.3 반환 항목

| 검정 | 반환 항목 |
|---|---|
| 공통 | 검정 유형, 테이블, 컬럼, 유의수준, 판정 문구(`decision`) |
| 일표본 | 기술통계, t값, 자유도, p-value, 평균의 신뢰구간, Cohen's d, Shapiro-Wilk 정규성 |
| 독립표본 | 그룹별 기술통계, Student와 Welch 결과, Levene 등분산 검정, 평균 차이의 신뢰구간(Welch), Cohen's d, 그룹별 정규성, 권장 검정 |
| 대응표본 | 쌍 개수, 조건별 기술통계, 평균 차이와 신뢰구간, t값, p-value, Cohen's dz, 차이값의 정규성 |

독립표본에서는 Levene 검정의 p-value가 유의수준 이상이면 Student를, 미만이면 Welch를 권장 검정으로 고르고, `decision`도 그 검정의 p-value 기준으로 판정합니다.

## 4.4 전체 코드

```sql
CREATE OR REPLACE PROCEDURE KWS_DB.STATISTICS.RUN_TTEST(
    TABLE_NAME   VARCHAR,
    VALUE_COL    VARCHAR,
    TEST_TYPE    VARCHAR DEFAULT 'independent',
    GROUP_COL    VARCHAR DEFAULT NULL,
    GROUP_A      VARCHAR DEFAULT NULL,
    GROUP_B      VARCHAR DEFAULT NULL,
    PAIR_ID_COL  VARCHAR DEFAULT NULL,
    MU0          FLOAT   DEFAULT 0,
    ALPHA        FLOAT   DEFAULT 0.05,
    FILTER_COL   VARCHAR DEFAULT NULL,
    FILTER_VALUE VARCHAR DEFAULT NULL
)
RETURNS VARIANT
LANGUAGE PYTHON
RUNTIME_VERSION = '3.11'
PACKAGES = ('snowflake-snowpark-python', 'pandas', 'numpy', 'scipy')
HANDLER = 'main'
COMMENT = 'One-sample / independent (Student, Welch) / paired t-test with Levene, Shapiro, Cohen d, CI'
AS
$$
import math
import re

import numpy as np
from scipy import stats
from snowflake.snowpark import Session
from snowflake.snowpark.functions import col

_IDENT = re.compile(r'^[A-Za-z_][A-Za-z0-9_$]*$')


def _check_ident(name, label, parts=1):
    if name is None or name == '':
        raise ValueError(f'{label} is required')
    pieces = name.split('.')
    if len(pieces) > parts or not all(_IDENT.match(p) for p in pieces):
        raise ValueError(f'invalid {label}: {name}')
    return name.upper()


def _f(x):
    if x is None:
        return None
    x = float(x)
    return None if math.isnan(x) or math.isinf(x) else float(f'{x:.6g}')


def _desc(a):
    n = len(a)
    return {
        'n': int(n),
        'mean': _f(np.mean(a)) if n else None,
        'std': _f(np.std(a, ddof=1)) if n > 1 else None,
        'median': _f(np.median(a)) if n else None,
        'min': _f(np.min(a)) if n else None,
        'max': _f(np.max(a)) if n else None,
    }


def _normality(a):
    if len(a) < 3:
        return None
    sample = a if len(a) <= 5000 else np.random.default_rng(0).choice(a, 5000, replace=False)
    w, p = stats.shapiro(sample)
    return {'test': 'Shapiro-Wilk', 'W': _f(w), 'p_value': _f(p)}


def _decision(p, alpha):
    if p is None:
        return None
    return '유의함 (귀무가설 기각)' if p < alpha else '유의하지 않음 (귀무가설 채택)'


def main(session: Session, table_name, value_col, test_type, group_col, group_a,
         group_b, pair_id_col, mu0, alpha, filter_col, filter_value):
    try:
        tbl = _check_ident(table_name, 'TABLE_NAME', parts=3)
        vcol = _check_ident(value_col, 'VALUE_COL')
        ttype = (test_type or 'independent').strip().lower()
        alpha = 0.05 if alpha is None else float(alpha)
        mu0 = 0.0 if mu0 is None else float(mu0)
        if ttype not in ('one_sample', 'independent', 'paired'):
            raise ValueError("TEST_TYPE must be one of: one_sample, independent, paired")

        df = session.table(tbl)
        cols = set(df.columns)
        needed = [vcol]

        def need(name, label):
            c = _check_ident(name, label)
            if c not in cols:
                raise ValueError(f'column not found: {c}')
            needed.append(c)
            return c

        if vcol not in cols:
            raise ValueError(f'column not found: {vcol}')
        if filter_col:
            fcol = need(filter_col, 'FILTER_COL')
            df = df.filter(col(fcol) == filter_value)

        result = {'test_type': ttype, 'table': tbl, 'value_col': vcol, 'alpha': alpha}
        if filter_col:
            result['filter'] = {filter_col.upper(): filter_value}

        if ttype == 'one_sample':
            a = df.filter(col(vcol).is_not_null()).select(vcol).to_pandas()[vcol].astype(float).to_numpy()
            if len(a) < 2:
                raise ValueError('need at least 2 non-null values')
            t, p = stats.ttest_1samp(a, mu0)
            se = stats.sem(a)
            dfree = len(a) - 1
            ci = stats.t.interval(1 - alpha, dfree, loc=np.mean(a), scale=se)
            result.update({
                'mu0': mu0,
                'descriptives': _desc(a),
                't_stat': _f(t), 'df': dfree, 'p_value': _f(p),
                'mean_ci': [_f(ci[0]), _f(ci[1])],
                'cohens_d': _f((np.mean(a) - mu0) / np.std(a, ddof=1)),
                'normality': _normality(a),
                'decision': _decision(_f(p), alpha),
            })
            return result

        gcol = need(group_col, 'GROUP_COL')
        if group_a is None or group_b is None:
            raise ValueError('GROUP_A and GROUP_B are required')

        if ttype == 'independent':
            pdf = (df.filter(col(gcol).isin([group_a, group_b]) & col(vcol).is_not_null())
                     .select(gcol, vcol).to_pandas())
            g = pdf[gcol].astype(str)
            a = pdf.loc[g == str(group_a), vcol].astype(float).to_numpy()
            b = pdf.loc[g == str(group_b), vcol].astype(float).to_numpy()
            if len(a) < 2 or len(b) < 2:
                raise ValueError('each group needs at least 2 non-null values')

            t_s, p_s = stats.ttest_ind(a, b, equal_var=True)
            t_w, p_w = stats.ttest_ind(a, b, equal_var=False)
            lev_w, lev_p = stats.levene(a, b, center='median')
            n1, n2 = len(a), len(b)
            v1, v2 = np.var(a, ddof=1), np.var(b, ddof=1)
            sp = math.sqrt(((n1 - 1) * v1 + (n2 - 1) * v2) / (n1 + n2 - 2))
            welch_df = (v1 / n1 + v2 / n2) ** 2 / ((v1 / n1) ** 2 / (n1 - 1) + (v2 / n2) ** 2 / (n2 - 1))
            diff = np.mean(a) - np.mean(b)
            se_w = math.sqrt(v1 / n1 + v2 / n2)
            ci = stats.t.interval(1 - alpha, welch_df, loc=diff, scale=se_w)
            equal_var = lev_p >= alpha
            p_rec = p_s if equal_var else p_w
            result.update({
                'group_col': gcol,
                'groups': {str(group_a): _desc(a), str(group_b): _desc(b)},
                'mean_diff_a_minus_b': _f(diff),
                'mean_diff_ci_welch': [_f(ci[0]), _f(ci[1])],
                'student': {'t_stat': _f(t_s), 'df': n1 + n2 - 2, 'p_value': _f(p_s)},
                'welch': {'t_stat': _f(t_w), 'df': _f(welch_df), 'p_value': _f(p_w)},
                'levene': {'W': _f(lev_w), 'p_value': _f(lev_p), 'equal_variance': bool(equal_var)},
                'normality': {str(group_a): _normality(a), str(group_b): _normality(b)},
                'cohens_d': _f(diff / sp) if sp > 0 else None,
                'recommended_test': 'student' if equal_var else 'welch',
                'decision': _decision(_f(p_rec), alpha),
            })
            return result

        # paired: GROUP_COL is the condition column (e.g. PERIOD), PAIR_ID_COL identifies the subject
        pcol = need(pair_id_col, 'PAIR_ID_COL')
        pdf = (df.filter(col(gcol).isin([group_a, group_b]) & col(vcol).is_not_null())
                 .select(pcol, gcol, vcol).to_pandas())
        pdf[gcol] = pdf[gcol].astype(str)
        wide = pdf.pivot_table(index=pcol, columns=gcol, values=vcol, aggfunc='mean')
        if str(group_a) not in wide.columns or str(group_b) not in wide.columns:
            raise ValueError('both conditions must exist in GROUP_COL')
        wide = wide[[str(group_a), str(group_b)]].dropna()
        a = wide[str(group_a)].to_numpy(dtype=float)
        b = wide[str(group_b)].to_numpy(dtype=float)
        if len(a) < 2:
            raise ValueError('need at least 2 complete pairs')
        d = a - b
        t, p = stats.ttest_rel(a, b)
        dfree = len(d) - 1
        ci = stats.t.interval(1 - alpha, dfree, loc=np.mean(d), scale=stats.sem(d))
        result.update({
            'group_col': gcol, 'pair_id_col': pcol,
            'n_pairs': int(len(d)),
            'conditions': {str(group_a): _desc(a), str(group_b): _desc(b)},
            'mean_diff_a_minus_b': _f(np.mean(d)),
            'mean_diff_ci': [_f(ci[0]), _f(ci[1])],
            't_stat': _f(t), 'df': dfree, 'p_value': _f(p),
            'cohens_dz': _f(np.mean(d) / np.std(d, ddof=1)) if np.std(d, ddof=1) > 0 else None,
            'normality_of_diff': _normality(d),
            'decision': _decision(_f(p), alpha),
        })
        return result
    except Exception as e:
        return {'error': str(e)}
$$;
```

# 5. 실행 및 결과 해석

> 프로시저는 결과를 VARIANT로 반환합니다. 클라이언트에 따라 `[object Object]`처럼 보이면, 호출 직후 아래 쿼리로 JSON 문자열을 확인할 수 있습니다.
> ```sql
> SELECT TO_JSON($1) FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));
> ```

## 5.1 독립표본 t검정: 프로모션 후 A그룹 vs B그룹

```sql
CALL KWS_DB.STATISTICS.RUN_TTEST(
  TABLE_NAME => 'KWS_DB.STATISTICS.SAMPLE_TTEST', VALUE_COL => 'PURCHASE_AMT',
  TEST_TYPE => 'independent', GROUP_COL => 'AB_GROUP', GROUP_A => 'A', GROUP_B => 'B',
  FILTER_COL => 'PERIOD', FILTER_VALUE => 'AFTER');
```

| 항목 | 값 |
|---|---|
| 평균 (A / B) | 51,323 / 56,723 |
| 평균 차이 (A − B) | −5,400 (95% CI: −8,188 ~ −2,613) |
| Levene p-value | 0.024 → 등분산 가정 기각, **Welch 권장** |
| Welch t / df / p | −3.82 / 187.2 / 0.000180 |
| (참고) Student t / df / p | −3.82 / 198 / 0.000177 |
| Cohen's d | −0.54 (중간 정도 효과) |
| 정규성 p (A / B) | 0.65 / 0.26 → 정규성 가정 만족 |
| 판정 | **유의함** |

**해석:** B그룹의 구매금액이 A그룹보다 평균 약 5,400원 높고, 통계적으로 유의합니다. 두 그룹의 분산이 달라서 Welch 검정 결과를 기준으로 판단했습니다.

## 5.2 대응표본 t검정: 같은 고객의 프로모션 전 vs 후

```sql
CALL KWS_DB.STATISTICS.RUN_TTEST(
  TABLE_NAME => 'KWS_DB.STATISTICS.SAMPLE_TTEST', VALUE_COL => 'PURCHASE_AMT',
  TEST_TYPE => 'paired', GROUP_COL => 'PERIOD', GROUP_A => 'AFTER', GROUP_B => 'BEFORE',
  PAIR_ID_COL => 'CUSTOMER_ID');
```

| 항목 | 값 |
|---|---|
| 쌍 개수 | 200 |
| 평균 (AFTER / BEFORE) | 54,023 / 49,023 |
| 평균 차이 (AFTER − BEFORE) | +5,000 (95% CI: 2,934 ~ 7,066) |
| t / df / p | 4.77 / 199 / 3.52 × 10⁻⁶ |
| Cohen's dz | 0.34 |
| 차이값 정규성 p | 0.83 |
| 판정 | **유의함** |

**해석:** 같은 고객 기준으로 프로모션 후 구매금액이 평균 약 5,000원 늘었고, 통계적으로 유의합니다.

## 5.3 일표본 t검정: 프로모션 전 평균이 50,000원인가

```sql
CALL KWS_DB.STATISTICS.RUN_TTEST(
  TABLE_NAME => 'KWS_DB.STATISTICS.SAMPLE_TTEST', VALUE_COL => 'PURCHASE_AMT',
  TEST_TYPE => 'one_sample', MU0 => 50000,
  FILTER_COL => 'PERIOD', FILTER_VALUE => 'BEFORE');
```

| 항목 | 값 |
|---|---|
| 표본 평균 | 49,023 (95% CI: 47,630 ~ 50,416) |
| t / df / p | −1.38 / 199 / 0.168 |
| Cohen's d | −0.10 |
| 판정 | **유의하지 않음** |

**해석:** 신뢰구간에 50,000이 포함되므로, 프로모션 전 평균이 50,000원과 다르다고 볼 근거가 없습니다. 설계한 모평균(50,000)과도 일치하는 결과입니다.

## 5.4 잘못된 입력 처리

```sql
CALL KWS_DB.STATISTICS.RUN_TTEST(
  TABLE_NAME => 'KWS_DB.STATISTICS.SAMPLE_TTEST; DROP TABLE X',
  VALUE_COL => 'PURCHASE_AMT', TEST_TYPE => 'one_sample');
-- {"error": "invalid TABLE_NAME: KWS_DB.STATISTICS.SAMPLE_TTEST; DROP TABLE X"}
```

식별자 검증에서 걸러지므로 쿼리는 실행되지 않고 오류 메시지만 반환됩니다.

## 5.5 설계값과 비교

| 검정 | 설계한 효과 | 측정 결과 | 결론 |
|---|---|---|---|
| 독립표본 (B − A) | +4,000 | +5,400 | 유의함 |
| 대응표본 (AFTER − BEFORE) | 약 +3,000* | +5,000 | 유의함 |
| 일표본 (BEFORE vs 50,000) | 0 | −977 | 유의하지 않음 |

\* AFTER의 기댓값은 A그룹 51,000, B그룹 55,000이고 두 그룹이 약 절반씩이므로 평균 약 53,000입니다. BEFORE 기댓값 50,000과 비교하면 약 3,000 차이입니다.

측정값과 설계값의 차이는 표본 200명 수준의 무작위 변동이며, 세 검정 모두 의도한 결론과 일치합니다.

# 6. 에이전트에 연결할 때 팁

Cortex Agent에 custom tool로 등록할 때는 **도구 설명(description)** 이 호출 정확도를 좌우합니다. 예시는 다음과 같습니다.

```text
두 그룹 또는 전후 비교에서 평균 차이가 통계적으로 유의한지 t검정으로 판단한다.
- 한 집단 평균을 특정 값과 비교: TEST_TYPE='one_sample', MU0에 기준값
- 서로 다른 두 집단 비교(A/B 테스트 등): TEST_TYPE='independent', GROUP_COL/GROUP_A/GROUP_B
- 같은 대상의 전후 비교: TEST_TYPE='paired', GROUP_COL=시점 컬럼, PAIR_ID_COL=대상 ID
- 특정 기간이나 조건만 분석할 때는 FILTER_COL/FILTER_VALUE 사용
결과의 decision, p_value, 평균 차이와 신뢰구간, 효과크기를 근거로 답하고,
독립표본은 recommended_test 결과를 우선 사용한다.
```

함께 챙겨야 할 것들입니다.

- **권한:** 에이전트를 사용하는 역할에 프로시저 USAGE 권한과 대상 테이블 SELECT 권한이 필요합니다.
- **역할 분담:** 기술통계나 빈도처럼 SQL로 되는 질문은 Cortex Analyst가, 검정은 이 도구가 처리하도록 에이전트 instructions에 적어두면 좋습니다.
- **대용량 데이터:** 프로시저가 데이터를 pandas로 읽어 들이므로, 큰 테이블은 `FILTER_COL`로 범위를 줄이거나 웨어하우스 크기를 고려해야 합니다.

# 7. 마무리

- 시맨틱 뷰만으로는 어려운 **t검정, p-value, 전제조건 검정**을 Python 저장 프로시저 하나로 해결했습니다.
- 세 가지 t검정을 **하나의 도구**로 묶어 에이전트가 쉽게 호출하도록 만들었습니다.
- 같은 방식으로 **카이제곱 검정, ANOVA, 상관 검정, 정규성 검정** 도구도 만들 수 있습니다.

---

**환경:** Snowflake, Python 3.11 런타임, `scipy` / `numpy` / `pandas` (Snowflake Anaconda 채널)
