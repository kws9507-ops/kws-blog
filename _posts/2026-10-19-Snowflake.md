---
title: "[Snowflake] Snowflake CoWork 에이전트를 위한 평균 비교 커스텀 도구 만들기"
date: 2026-10-19 10:00:00 +0900
categories:
  - Snowflake
tags:
  - Snowflake
  - t검정
  - Z검정
  - 비모수 검정
---

> Cortex Analyst(시맨틱 뷰)만으로는 어려운 **t검정, Z검정, 비모수 검정과 p-value 계산**을 Python 저장 프로시저 하나로 구현해, Snowflake CoWork 에이전트의 커스텀 도구로 쓸 수 있게 만드는 과정을 정리했습니다.

# 목차
1. [왜 커스텀 도구가 필요한가](#1-왜-커스텀-도구가-필요한가)
2. [전체 구성](#2-전체-구성)
3. [샘플 데이터 만들기](#3-샘플-데이터-만들기)
4. [평균 비교 프로시저 만들기](#4-평균-비교-프로시저-만들기)
5. [실행 및 결과 해석](#5-실행-및-결과-해석)
6. [에이전트에 연결할 때 팁](#6-에이전트에-연결할-때-팁)
7. [마무리](#7-마무리)

---

# 1. 왜 커스텀 도구가 필요한가

Snowflake CoWork(구 Snowflake Intelligence)의 Cortex Analyst 도구는 자연어 질문을 **SQL로 바꿔 실행**합니다. 평균이나 표준편차 같은 기술통계는 시맨틱 뷰만으로 충분하지만, 평균 차이를 **검정**하는 일은 다릅니다.

| 항목 | SQL만으로 처리할 때 |
|---|---|
| t값, z값 계산 | `AVG`, `STDDEV_SAMP`, `COUNT`, `SQRT`로 가능 |
| t검정 판정 | 임계값이 자유도에 따라 달라져서 고정값을 쓸 수 없음 |
| p-value | t분포·정규분포의 누적분포함수가 Snowflake SQL에 기본 제공되지 않음* |
| Welch 자유도 | 공식이 복잡해서 LLM이 매번 정확하게 생성하기 어려움 |
| 비모수 검정 | 순위 계산, 동점 처리, 정확 분포가 필요해서 SQL로 구현하기 어려움 |
| 전제조건 확인 | 정규성(Shapiro), 등분산(Levene) 검정은 SQL로 구현하기 어려움 |

\* 작성 시점 기준으로 확인한 내용입니다. 이후 Snowflake에 관련 함수가 추가되었을 수 있으니 공식 문서를 함께 확인하세요.

그래서 `scipy.stats`를 쓰는 **Python 저장 프로시저**를 만들고, 이를 에이전트의 custom tool로 등록하는 방식을 택했습니다.

## 이 도구가 다루는 질문: "1~2집단의 평균(위치) 비교"

| 상황 | 모수 검정 | 비모수 대안 |
|---|---|---|
| 한 집단 평균 vs 기준값 | 일표본 t검정 / 일표본 Z검정 | Wilcoxon 부호순위 검정, 부호검정 |
| 서로 다른 두 집단 | 독립표본 t검정(Student, Welch) / 두 표본 Z검정 | Mann-Whitney U 검정 |
| 같은 대상의 전후(대응) | 대응표본 t검정 | Wilcoxon 부호순위 검정, 부호검정 |

세 집단 이상의 평균 비교는 분산분석 도구(`RUN_ANOVA`)에서 다룹니다.

### t검정과 Z검정의 차이
- **Z검정:** 모집단 표준편차(σ)를 **알고 있을 때** 씁니다. 예를 들어 과거 수년간의 데이터로 σ가 알려진 공정을 관리하는 경우입니다.
- **t검정:** σ를 모르고 표본 표준편차로 추정할 때 씁니다. 실무에서는 대부분 이 경우입니다.
- 표본이 크면(대략 n ≥ 30) 두 검정의 결과가 거의 같아집니다.

### 비모수 검정이 필요한 경우
t검정은 데이터가 정규분포를 따른다고 가정합니다. **표본이 작고(n < 30) 분포가 치우쳐 있거나 이상치가 있으면** t검정 결과를 믿기 어렵습니다. 이럴 때는 값 대신 **순위**를 쓰는 비모수 검정이 더 안정적입니다.

# 2. 전체 구성

```
KWS_DB
└── STATISTICS (schema)
    ├── SAMPLE_MEAN        -- 테스트용 샘플 테이블
    └── RUN_MEAN_TEST(...) -- 평균 비교 Python 저장 프로시저 (VARIANT 반환)
```

동작 흐름은 다음과 같습니다.

```
사용자 질문 (CoWork)
  → 에이전트가 RUN_MEAN_TEST 도구를 선택하고 TEST_TYPE과 파라미터를 채움
  → 프로시저가 데이터를 읽어 scipy로 검정
  → 정규성 결과에 따라 모수 / 비모수 중 권장 검정을 판단
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

이렇게 구성하면 하나의 테이블로 이 도구의 모든 검정을 테스트할 수 있습니다.

| 검정 | 이 데이터에서 묻는 질문 |
|---|---|
| 일표본 t / Z / Wilcoxon / 부호검정 | 프로모션 전 구매금액이 50,000원과 다른가? |
| 독립표본 t / Z / Mann-Whitney | 프로모션 후 A그룹과 B그룹의 구매금액이 다른가? |
| 대응표본 t / Wilcoxon / 부호검정 | 같은 고객의 구매금액이 프로모션 전후로 달라졌는가? |

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

즉 **B그룹 프로모션 효과가 약 4,000원**이 되도록 의도적으로 설계했습니다. 또 BEFORE의 모표준편차를 10,000으로 정했기 때문에, Z검정에 쓸 "알려진 σ"로 활용할 수 있습니다.

## 3.4 생성 SQL

```sql
CREATE OR REPLACE TABLE KWS_DB.STATISTICS.SAMPLE_MEAN AS
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

# 4. 평균 비교 프로시저 만들기

## 4.1 설계 포인트

- **8가지 검정을 하나의 도구로:** `TEST_TYPE` 파라미터로 검정을 선택합니다. "두 그룹 평균이 다른가?"라는 같은 질문에 쓰이는 검정들이라, 에이전트가 고를 도구가 하나뿐이면 호출이 단순해집니다.
- **모수 / 비모수 자동 권장:** t검정을 실행하면 비모수 대안도 함께 계산합니다. 표본이 작고(n < 30) 정규성이 깨지면 `recommended_test`를 비모수 검정으로 바꾸고, `decision`도 그 결과를 기준으로 판정합니다. n ≥ 30이면 중심극한정리에 따라 t검정을 유지합니다.
- **Z검정의 안전장치:** σ(`SIGMA`)를 주지 않았는데 n < 30이면 오류를 반환하고 t검정을 쓰도록 안내합니다. n ≥ 30이면 표본 표준편차로 근사하고, 근사했다는 사실을 `sigma_basis`에 표시합니다.
- **Wilcoxon과 부호검정은 두 가지 모드:** `PAIR_ID_COL`을 주면 대응표본, 주지 않으면 `MU0` 기준의 일표본 검정으로 동작합니다.
- **효과크기:** Cohen's d(모수), 순위이연상관 r과 우월확률(비모수)을 함께 반환해서 "얼마나 차이 나는가"까지 설명할 수 있게 합니다.
- **입력 검증:** 테이블명과 컬럼명을 정규식으로 검증하고, 실제 테이블에 있는 컬럼인지 확인합니다. 쿼리는 문자열을 이어 붙이지 않고 Snowpark DataFrame API로 구성해서 SQL 인젝션을 막습니다.
- **원인을 알 수 있는 오류 메시지:** 필터 결과가 0행이거나 그룹 값이 없을 때 그 원인을 구체적으로 알려줍니다. 예외는 `{"error": "..."}` 형태로 반환되어, 에이전트가 사용자에게 설명할 수 있습니다.
- **정확한 수치 표현:** 결과는 유효숫자 6자리로 반올림합니다. 소수 자릿수로 반올림하면 `3.52e-6` 같은 아주 작은 p-value가 뭉개지기 때문입니다.

## 4.2 파라미터

| 파라미터 | 기본값 | 설명 |
|---|---|---|
| `TABLE_NAME` | (필수) | 분석할 테이블. `DB.SCHEMA.TABLE` 형식 |
| `VALUE_COL` | (필수) | 검정할 수치 컬럼 |
| `TEST_TYPE` | `independent` | 아래 표 참고 |
| `GROUP_COL` | NULL | 그룹 컬럼. 대응표본에서는 조건 컬럼(예: `PERIOD`) |
| `GROUP_A`, `GROUP_B` | NULL | 비교할 두 그룹(조건) 값. 차이는 A − B로 계산 |
| `PAIR_ID_COL` | NULL | 대응표본에서 같은 대상을 식별하는 키 |
| `MU0` | 0 | 일표본 검정의 기준값 |
| `SIGMA` | NULL | Z검정의 모표준편차. 두 표본 Z검정에서는 A그룹의 σ |
| `SIGMA_B` | NULL | 두 표본 Z검정에서 B그룹의 σ. 생략하면 `SIGMA`와 같다고 봄 |
| `ALPHA` | 0.05 | 유의수준 |
| `FILTER_COL`, `FILTER_VALUE` | NULL | 선택 사항. `FILTER_COL = FILTER_VALUE` 조건으로 데이터를 좁힘 |

**`TEST_TYPE` 값**

| 값 | 검정 | 필요한 파라미터 |
|---|---|---|
| `one_sample` | 일표본 t검정 (+ Wilcoxon 대안) | `MU0` |
| `independent` | 독립표본 t검정 Student/Welch (+ Mann-Whitney 대안) | `GROUP_COL`, `GROUP_A`, `GROUP_B` |
| `paired` | 대응표본 t검정 (+ Wilcoxon 대안) | `GROUP_COL`, `GROUP_A`, `GROUP_B`, `PAIR_ID_COL` |
| `z_one_sample` | 일표본 Z검정 | `MU0`, `SIGMA` (n ≥ 30이면 생략 가능) |
| `z_two_sample` | 두 표본 Z검정 | `GROUP_COL`, `GROUP_A`, `GROUP_B`, `SIGMA`(, `SIGMA_B`) |
| `mann_whitney` | Mann-Whitney U 검정 | `GROUP_COL`, `GROUP_A`, `GROUP_B` |
| `wilcoxon` | Wilcoxon 부호순위 검정 | 대응: `GROUP_COL`, `GROUP_A`, `GROUP_B`, `PAIR_ID_COL` / 일표본: `MU0` |
| `sign` | 부호검정 | `wilcoxon`과 같음 |

## 4.3 반환 항목

| 검정 | 반환 항목 |
|---|---|
| 공통 | 검정 유형, 테이블, 컬럼, 유의수준, scipy 버전, 판정 문구(`decision`) |
| t검정 3종 | 기술통계, t값, 자유도, p-value, 신뢰구간, Cohen's d(dz), 정규성, **비모수 대안 결과**, `normality_ok`, `recommended_test` |
| 독립표본 t | 위 항목에 더해 Student와 Welch 결과, Levene 등분산 검정 |
| Z검정 | 사용한 σ와 그 근거(`sigma_basis`), z값, p-value, 임계 z값, 신뢰구간 |
| Mann-Whitney | U, p-value, 우월확률 P(A > B), 순위이연상관 r, 중앙값 차이 |
| Wilcoxon | W, p-value, 0이 아닌 차이의 개수, 순위이연상관 r |
| 부호검정 | 양수·음수 개수, 제외된 동점 수, p-value |

**권장 검정 판정 규칙**

| 조건 | 일표본 / 대응표본 | 독립표본 |
|---|---|---|
| 정규성 만족 (또는 n ≥ 30) + 등분산 | t검정 | Student t |
| 정규성 만족 (또는 n ≥ 30) + 이분산 | t검정 | Welch t |
| n < 30이고 정규성 위반 | Wilcoxon | Mann-Whitney |

**결과 읽는 법**

| 지표 | 해석 기준 |
|---|---|
| Cohen's d | 0.2 작음, 0.5 중간, 0.8 큼 |
| 순위이연상관 r | 0.1 작음, 0.3 중간, 0.5 큼 (부호는 방향) |
| 우월확률 P(A > B) | A에서 무작위로 뽑은 값이 B보다 클 확률. 0.5면 차이 없음 |

## 4.4 전체 코드

```sql
CREATE OR REPLACE PROCEDURE KWS_DB.STATISTICS.RUN_MEAN_TEST(
    TABLE_NAME   VARCHAR,
    VALUE_COL    VARCHAR,
    TEST_TYPE    VARCHAR DEFAULT 'independent',
    GROUP_COL    VARCHAR DEFAULT NULL,
    GROUP_A      VARCHAR DEFAULT NULL,
    GROUP_B      VARCHAR DEFAULT NULL,
    PAIR_ID_COL  VARCHAR DEFAULT NULL,
    MU0          FLOAT   DEFAULT 0,
    SIGMA        FLOAT   DEFAULT NULL,
    SIGMA_B      FLOAT   DEFAULT NULL,
    ALPHA        FLOAT   DEFAULT 0.05,
    FILTER_COL   VARCHAR DEFAULT NULL,
    FILTER_VALUE VARCHAR DEFAULT NULL
)
RETURNS VARIANT
LANGUAGE PYTHON
RUNTIME_VERSION = '3.11'
PACKAGES = ('snowflake-snowpark-python', 'pandas', 'numpy', 'scipy')
HANDLER = 'main'
COMMENT = 'Mean/location tests for 1-2 groups: t (one-sample, Student/Welch, paired), Z (mean), Mann-Whitney U, Wilcoxon signed-rank, sign test'
AS
$$
import math
import re

import numpy as np
import scipy
from scipy import stats
from snowflake.snowpark import Session
from snowflake.snowpark.functions import col

_IDENT = re.compile(r'^[A-Za-z_][A-Za-z0-9_$]*$')
TYPES = ('one_sample', 'independent', 'paired', 'z_one_sample', 'z_two_sample',
         'mann_whitney', 'wilcoxon', 'sign')


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
    if len(a) < 3 or np.std(a) == 0:
        return None
    sample = a if len(a) <= 5000 else np.random.default_rng(0).choice(a, 5000, replace=False)
    w, p = stats.shapiro(sample)
    return {'test': 'Shapiro-Wilk', 'W': _f(w), 'p_value': _f(p)}


def _normal_enough(a, alpha):
    # t-tests are robust for n >= 30 (CLT); otherwise require Shapiro-Wilk p >= alpha
    if len(a) >= 30:
        return True
    nm = _normality(a)
    return nm is None or nm['p_value'] >= alpha


def _decision(p, alpha):
    if p is None:
        return None
    return '유의함 (귀무가설 기각)' if p < alpha else '유의하지 않음 (귀무가설 채택)'


def _wilcoxon_1s(d):
    d = d[d != 0]
    n = len(d)
    if n < 1:
        raise ValueError('all differences are zero; Wilcoxon test is undefined')
    res = stats.wilcoxon(d)
    ranks = stats.rankdata(np.abs(d))
    w_plus, w_minus = ranks[d > 0].sum(), ranks[d < 0].sum()
    r_rb = (w_plus - w_minus) / (n * (n + 1) / 2)
    return {'W': _f(res.statistic), 'p_value': _f(res.pvalue), 'n_nonzero': int(n),
            'rank_biserial_r': _f(r_rb)}


def _sign(d):
    d = d[d != 0]
    n = len(d)
    if n < 1:
        raise ValueError('all differences are zero; sign test is undefined')
    k = int((d > 0).sum())
    res = stats.binomtest(k, n, 0.5)
    return {'n_positive': k, 'n_negative': int(n - k), 'n_ties_dropped': None,
            'p_value': _f(res.pvalue)}


def _mann_whitney(a, b):
    res = stats.mannwhitneyu(a, b, alternative='two-sided')
    u1 = res.statistic
    n1, n2 = len(a), len(b)
    return {'U': _f(u1), 'p_value': _f(res.pvalue),
            'prob_a_greater': _f(u1 / (n1 * n2)),
            'rank_biserial_r': _f(2 * u1 / (n1 * n2) - 1),
            'median_diff_a_minus_b': _f(np.median(a) - np.median(b))}


def main(session: Session, table_name, value_col, test_type, group_col, group_a, group_b,
         pair_id_col, mu0, sigma, sigma_b, alpha, filter_col, filter_value):
    try:
        tbl = _check_ident(table_name, 'TABLE_NAME', parts=3)
        vcol = _check_ident(value_col, 'VALUE_COL')
        ttype = (test_type or 'independent').strip().lower()
        alpha = 0.05 if alpha is None else float(alpha)
        mu0 = 0.0 if mu0 is None else float(mu0)
        if ttype not in TYPES:
            raise ValueError('TEST_TYPE must be one of: ' + ', '.join(TYPES))

        df = session.table(tbl)
        cols = set(df.columns)

        def need(name, label):
            c = _check_ident(name, label)
            if c not in cols:
                raise ValueError(f'column not found: {c}')
            return c

        if vcol not in cols:
            raise ValueError(f'column not found: {vcol}')
        if filter_col:
            fcol = need(filter_col, 'FILTER_COL')
            df = df.filter(col(fcol) == filter_value)

        result = {'test_type': ttype, 'table': tbl, 'value_col': vcol, 'alpha': alpha,
                  'scipy_version': scipy.__version__}
        if filter_col:
            result['filter'] = {filter_col.upper(): filter_value}

        # wilcoxon / sign: paired when PAIR_ID_COL is given, otherwise one-sample against MU0
        paired_mode = ttype == 'paired' or (ttype in ('wilcoxon', 'sign') and pair_id_col)
        one_mode = ttype in ('one_sample', 'z_one_sample') or (ttype in ('wilcoxon', 'sign') and not pair_id_col)

        def load_one():
            a = (df.filter(col(vcol).is_not_null()).select(vcol).to_pandas()[vcol]
                   .to_numpy(dtype=float))
            if len(a) == 0:
                raise ValueError('no rows match the given conditions (check FILTER_COL/FILTER_VALUE)')
            if len(a) < 2:
                raise ValueError('need at least 2 non-null values')
            return a

        def load_two():
            gcol = need(group_col, 'GROUP_COL')
            if group_a is None or group_b is None:
                raise ValueError('GROUP_A and GROUP_B are required')
            pdf = (df.filter(col(gcol).isin([group_a, group_b]) & col(vcol).is_not_null())
                     .select(gcol, vcol).to_pandas())
            g = pdf[gcol].astype(str)
            a = pdf.loc[g == str(group_a), vcol].to_numpy(dtype=float)
            b = pdf.loc[g == str(group_b), vcol].to_numpy(dtype=float)
            missing = [v for v, arr in ((group_a, a), (group_b, b)) if len(arr) == 0]
            if missing:
                raise ValueError(f'no data for group value(s) {missing} in {gcol} (check GROUP_A/GROUP_B and filters)')
            if len(a) < 2 or len(b) < 2:
                raise ValueError('each group needs at least 2 non-null values')
            result['group_col'] = gcol
            return a, b

        def load_pairs():
            gcol = need(group_col, 'GROUP_COL')
            pcol = need(pair_id_col, 'PAIR_ID_COL')
            if group_a is None or group_b is None:
                raise ValueError('GROUP_A and GROUP_B are required')
            pdf = (df.filter(col(gcol).isin([group_a, group_b]) & col(vcol).is_not_null())
                     .select(pcol, gcol, vcol).to_pandas())
            if pdf.empty:
                raise ValueError(f'no data for GROUP_A/GROUP_B in {gcol} (check values and filters)')
            pdf[gcol] = pdf[gcol].astype(str)
            wide = pdf.pivot_table(index=pcol, columns=gcol, values=vcol, aggfunc='mean')
            for v in (str(group_a), str(group_b)):
                if v not in wide.columns:
                    raise ValueError(f'no data for condition {v!r} in {gcol}')
            wide = wide[[str(group_a), str(group_b)]].dropna()
            if len(wide) < 2:
                raise ValueError('need at least 2 complete pairs')
            result.update({'group_col': gcol, 'pair_id_col': pcol})
            return wide[str(group_a)].to_numpy(dtype=float), wide[str(group_b)].to_numpy(dtype=float)

        # ---------- one-sample family ----------
        if one_mode:
            a = load_one()
            result.update({'mu0': mu0, 'descriptives': _desc(a), 'normality': _normality(a)})
            if ttype == 'z_one_sample':
                if sigma is None:
                    if len(a) < 30:
                        raise ValueError('SIGMA (population std) is required for z-test when n < 30; use one_sample t-test')
                    s, basis = float(np.std(a, ddof=1)), 'sample_std (large-sample approximation)'
                else:
                    s, basis = float(sigma), 'given'
                se = s / math.sqrt(len(a))
                z = (np.mean(a) - mu0) / se
                p = 2 * stats.norm.sf(abs(z))
                zc = stats.norm.ppf(1 - alpha / 2)
                result.update({'sigma': _f(s), 'sigma_basis': basis, 'z_stat': _f(z), 'p_value': _f(p),
                               'critical_z': _f(zc),
                               'mean_ci': [_f(np.mean(a) - zc * se), _f(np.mean(a) + zc * se)],
                               'decision': _decision(_f(p), alpha)})
                return result
            if ttype == 'wilcoxon':
                w = _wilcoxon_1s(a - mu0)
                result.update({'wilcoxon': w, 'decision': _decision(w['p_value'], alpha)})
                return result
            if ttype == 'sign':
                s_ = _sign(a - mu0)
                s_['n_ties_dropped'] = int((a == mu0).sum())
                result.update({'sign_test': s_, 'median_vs_mu0': _f(np.median(a) - mu0),
                               'decision': _decision(s_['p_value'], alpha)})
                return result
            t, p = stats.ttest_1samp(a, mu0)
            dfree = len(a) - 1
            ci = stats.t.interval(1 - alpha, dfree, loc=np.mean(a), scale=stats.sem(a))
            normal_ok = _normal_enough(a, alpha)
            alt = _wilcoxon_1s(a - mu0) if np.any(a != mu0) else None
            rec = 'one_sample_t' if normal_ok or alt is None else 'wilcoxon'
            p_rec = p if rec == 'one_sample_t' else alt['p_value']
            result.update({'t_stat': _f(t), 'df': dfree, 'p_value': _f(p),
                           'mean_ci': [_f(ci[0]), _f(ci[1])],
                           'cohens_d': _f((np.mean(a) - mu0) / np.std(a, ddof=1)),
                           'nonparametric_wilcoxon': alt, 'normality_ok': bool(normal_ok),
                           'recommended_test': rec, 'decision': _decision(_f(p_rec), alpha)})
            return result

        # ---------- paired family ----------
        if paired_mode:
            a, b = load_pairs()
            d = a - b
            result.update({'n_pairs': int(len(d)),
                           'conditions': {str(group_a): _desc(a), str(group_b): _desc(b)},
                           'mean_diff_a_minus_b': _f(np.mean(d)),
                           'median_diff_a_minus_b': _f(np.median(d)),
                           'normality_of_diff': _normality(d)})
            if ttype == 'wilcoxon':
                w = _wilcoxon_1s(d)
                result.update({'wilcoxon': w, 'decision': _decision(w['p_value'], alpha)})
                return result
            if ttype == 'sign':
                s_ = _sign(d)
                s_['n_ties_dropped'] = int((d == 0).sum())
                result.update({'sign_test': s_, 'decision': _decision(s_['p_value'], alpha)})
                return result
            t, p = stats.ttest_rel(a, b)
            dfree = len(d) - 1
            ci = stats.t.interval(1 - alpha, dfree, loc=np.mean(d), scale=stats.sem(d))
            normal_ok = _normal_enough(d, alpha)
            alt = _wilcoxon_1s(d) if np.any(d != 0) else None
            rec = 'paired_t' if normal_ok or alt is None else 'wilcoxon'
            p_rec = p if rec == 'paired_t' else alt['p_value']
            sd_d = np.std(d, ddof=1)
            result.update({'t_stat': _f(t), 'df': dfree, 'p_value': _f(p),
                           'mean_diff_ci': [_f(ci[0]), _f(ci[1])],
                           'cohens_dz': _f(np.mean(d) / sd_d) if sd_d > 0 else None,
                           'nonparametric_wilcoxon': alt, 'normality_ok': bool(normal_ok),
                           'recommended_test': rec, 'decision': _decision(_f(p_rec), alpha)})
            return result

        # ---------- two independent groups ----------
        a, b = load_two()
        n1, n2 = len(a), len(b)
        diff = np.mean(a) - np.mean(b)
        result.update({'groups': {str(group_a): _desc(a), str(group_b): _desc(b)},
                       'mean_diff_a_minus_b': _f(diff),
                       'normality': {str(group_a): _normality(a), str(group_b): _normality(b)}})

        if ttype == 'mann_whitney':
            mw = _mann_whitney(a, b)
            result.update({'mann_whitney': mw, 'decision': _decision(mw['p_value'], alpha)})
            return result

        if ttype == 'z_two_sample':
            if sigma is None:
                if min(n1, n2) < 30:
                    raise ValueError('SIGMA is required for z-test when a group has n < 30; use independent t-test')
                s1, s2 = float(np.std(a, ddof=1)), float(np.std(b, ddof=1))
                basis = 'sample_std (large-sample approximation)'
            else:
                s1 = float(sigma)
                s2 = float(sigma_b) if sigma_b is not None else s1
                basis = 'given'
            se = math.sqrt(s1 ** 2 / n1 + s2 ** 2 / n2)
            z = diff / se
            p = 2 * stats.norm.sf(abs(z))
            zc = stats.norm.ppf(1 - alpha / 2)
            result.update({'sigma_a': _f(s1), 'sigma_b': _f(s2), 'sigma_basis': basis,
                           'z_stat': _f(z), 'p_value': _f(p), 'critical_z': _f(zc),
                           'mean_diff_ci': [_f(diff - zc * se), _f(diff + zc * se)],
                           'decision': _decision(_f(p), alpha)})
            return result

        t_s, p_s = stats.ttest_ind(a, b, equal_var=True)
        t_w, p_w = stats.ttest_ind(a, b, equal_var=False)
        lev_w, lev_p = stats.levene(a, b, center='median')
        v1, v2 = np.var(a, ddof=1), np.var(b, ddof=1)
        sp = math.sqrt(((n1 - 1) * v1 + (n2 - 1) * v2) / (n1 + n2 - 2))
        welch_df = (v1 / n1 + v2 / n2) ** 2 / ((v1 / n1) ** 2 / (n1 - 1) + (v2 / n2) ** 2 / (n2 - 1))
        ci = stats.t.interval(1 - alpha, welch_df, loc=diff, scale=math.sqrt(v1 / n1 + v2 / n2))
        equal_var = lev_p >= alpha
        normal_ok = _normal_enough(a, alpha) and _normal_enough(b, alpha)
        mw = _mann_whitney(a, b)
        if not normal_ok:
            rec, p_rec = 'mann_whitney', mw['p_value']
        elif equal_var:
            rec, p_rec = 'student', p_s
        else:
            rec, p_rec = 'welch', p_w
        result.update({
            'mean_diff_ci_welch': [_f(ci[0]), _f(ci[1])],
            'student': {'t_stat': _f(t_s), 'df': n1 + n2 - 2, 'p_value': _f(p_s)},
            'welch': {'t_stat': _f(t_w), 'df': _f(welch_df), 'p_value': _f(p_w)},
            'levene': {'W': _f(lev_w), 'p_value': _f(lev_p), 'equal_variance': bool(equal_var)},
            'cohens_d': _f(diff / sp) if sp > 0 else None,
            'nonparametric_mann_whitney': mw,
            'normality_ok': bool(normal_ok),
            'recommended_test': rec,
            'decision': _decision(_f(p_rec), alpha),
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

## 5.1 독립표본: 프로모션 후 A그룹 vs B그룹

```sql
CALL KWS_DB.STATISTICS.RUN_MEAN_TEST(
  TABLE_NAME => 'KWS_DB.STATISTICS.SAMPLE_MEAN', VALUE_COL => 'PURCHASE_AMT',
  TEST_TYPE => 'independent', GROUP_COL => 'AB_GROUP', GROUP_A => 'A', GROUP_B => 'B',
  FILTER_COL => 'PERIOD', FILTER_VALUE => 'AFTER');
```

| 항목 | 값 |
|---|---|
| 평균 (A / B) | 51,323 / 56,723 |
| 평균 차이 (A − B) | −5,400 (95% CI: −8,188 ~ −2,613) |
| Levene p-value | 0.024 → 등분산 가정 기각 |
| Student t / df / p | −3.82 / 198 / 0.000177 |
| **Welch t / df / p** | **−3.82 / 187.2 / 0.000180** |
| Cohen's d | −0.54 (중간 정도 효과) |
| Mann-Whitney (비모수 대안) | U = 3,493, p = 0.000232, P(A > B) = 0.35, r = −0.30 |
| 권장 검정 | Welch (n ≥ 30이라 정규성 충족, 분산이 달라 Welch) |
| 판정 | **유의함** |

**해석:** B그룹의 구매금액이 A그룹보다 평균 약 5,400원 높고, 통계적으로 유의합니다. 비모수 검정인 Mann-Whitney도 같은 결론이어서 결과가 견고합니다. A그룹 고객이 B그룹 고객보다 구매금액이 클 확률은 35%입니다.

**두 표본 Z검정과 비교:**
```sql
CALL KWS_DB.STATISTICS.RUN_MEAN_TEST(
  TABLE_NAME => 'KWS_DB.STATISTICS.SAMPLE_MEAN', VALUE_COL => 'PURCHASE_AMT',
  TEST_TYPE => 'z_two_sample', GROUP_COL => 'AB_GROUP', GROUP_A => 'A', GROUP_B => 'B',
  FILTER_COL => 'PERIOD', FILTER_VALUE => 'AFTER');
```
σ를 주지 않았지만 두 그룹 모두 n = 100이라 표본 표준편차로 근사합니다(`sigma_basis = sample_std (large-sample approximation)`). 결과는 z = −3.82, p = 0.000133으로, 같은 통계량을 정규분포 기준으로 판정한 것이라 Welch t와 거의 같습니다. **표본이 크면 Z검정과 t검정의 결과가 수렴한다**는 점을 보여줍니다.

## 5.2 대응표본: 같은 고객의 프로모션 전 vs 후

```sql
CALL KWS_DB.STATISTICS.RUN_MEAN_TEST(
  TABLE_NAME => 'KWS_DB.STATISTICS.SAMPLE_MEAN', VALUE_COL => 'PURCHASE_AMT',
  TEST_TYPE => 'paired', GROUP_COL => 'PERIOD', GROUP_A => 'AFTER', GROUP_B => 'BEFORE',
  PAIR_ID_COL => 'CUSTOMER_ID');
```

| 검정 | 통계량 | p-value | 판정 |
|---|---|---|---|
| **대응표본 t** | t = 4.77, df = 199 | 3.52 × 10⁻⁶ | 유의함 |
| Wilcoxon 부호순위 (`TEST_TYPE => 'wilcoxon'`) | W = 6,474, r = 0.36 | 1.28 × 10⁻⁵ | 유의함 |
| 부호검정 (`TEST_TYPE => 'sign'`) | 증가 120명 / 감소 80명 | 0.00569 | 유의함 |

평균 차이(AFTER − BEFORE)는 +5,000원(95% CI: 2,934 ~ 7,066)이고 Cohen's dz는 0.34입니다.

**해석:** 같은 고객 기준으로 프로모션 후 구매금액이 늘었고, 세 검정 모두 유의합니다. 검출력은 **t검정 > Wilcoxon > 부호검정** 순으로 약해지는데, 사용하는 정보량이 값 → 순위 → 부호(증감 방향)로 줄어들기 때문입니다. 데이터가 정규분포에 가까우면 t검정이 가장 민감하고, 부호검정은 증가했는지 감소했는지만 보므로 가장 보수적입니다.

## 5.3 일표본: 프로모션 전 평균이 50,000원인가

| 검정 | 호출 | 결과 | 판정 |
|---|---|---|---|
| **일표본 t** | `TEST_TYPE => 'one_sample', MU0 => 50000` | t = −1.38, df = 199, p = 0.168, 평균 95% CI: 47,630 ~ 50,416 | 유의하지 않음 |
| 일표본 Z | `TEST_TYPE => 'z_one_sample', MU0 => 50000, SIGMA => 10000` | z = −1.38, p = 0.167, 평균 95% CI: 47,637 ~ 50,409 | 유의하지 않음 |
| Wilcoxon | `TEST_TYPE => 'wilcoxon', MU0 => 50000` | W = 9,220, p = 0.311 | 유의하지 않음 |

모든 호출에 `FILTER_COL => 'PERIOD', FILTER_VALUE => 'BEFORE'`를 함께 지정합니다.

**해석:** 신뢰구간에 50,000이 포함되므로, 프로모션 전 평균이 50,000원과 다르다고 볼 근거가 없습니다. Z검정에는 데이터를 만들 때 설정한 모표준편차(10,000)를 σ로 넣었습니다.

## 5.4 비모수 검정으로 자동 전환되는 경우

이상치가 섞인 소표본으로 자동 전환 기능을 확인했습니다. 그룹당 10개씩이고, 각 그룹에 극단값이 하나씩 들어 있습니다.

```
A: 1.0, 1.2, 1.1, 1.3, 1.0, 1.4, 1.2, 1.1, 1.0, 25.0
B: 3.0, 3.5, 2.8, 3.1, 3.3, 2.9, 3.2, 3.4, 3.0, 40.0
```

| 항목 | 결과 |
|---|---|
| 정규성 (`normality_ok`) | false (n < 30이고 Shapiro-Wilk 기각) |
| Welch t | p = 0.465 → 유의하지 않음 |
| Mann-Whitney | p = 0.00214 → 유의함 |
| `recommended_test` | **mann_whitney** |
| `decision` | **유의함** |

**해석:** 값 대부분은 B가 A보다 확실히 큰데, 극단값(25.0, 40.0) 때문에 표준편차가 커져서 t검정이 차이를 잡아내지 못했습니다. 순위를 쓰는 Mann-Whitney는 이상치의 영향을 받지 않아 차이를 정확히 잡아냈습니다. 도구가 이 상황을 감지하고 **스스로 권장 검정을 바꿔서 판정**합니다. 같은 조건의 일표본 검정에서도 t검정(p = 0.317) 대신 Wilcoxon(p = 0.0156)이 권장되었습니다.

## 5.5 잘못된 입력 처리

| 입력 | 반환되는 오류 |
|---|---|
| 테이블명에 `; DROP TABLE X` | `invalid TABLE_NAME: ...` (쿼리 실행 안 됨) |
| 존재하지 않는 그룹 값 `GROUP_B => 'Z'` | `no data for group value(s) ['Z'] in AB_GROUP (check GROUP_A/GROUP_B and filters)` |
| 필터 결과 0행 | `no rows match the given conditions (check FILTER_COL/FILTER_VALUE)` |
| n < 30인데 σ 없이 Z검정 | `SIGMA (population std) is required for z-test when n < 30; use one_sample t-test` |
| 지원하지 않는 `TEST_TYPE` | 사용 가능한 값 목록을 함께 반환 |

## 5.6 SQL로 직접 계산한 값과 교차검증

프로시저를 거치지 않고 SQL 공식으로 다시 계산해서 비교했습니다.

| 지표 | 프로시저 | SQL 직접 계산 |
|---|---|---|
| 일표본 t | −1.38342 | −1.383423 |
| Student t | −3.82158 | −3.821580 |
| Welch 자유도 | 187.190 | 187.190 |
| 대응표본 t | 4.77222 | 4.772221 |
| 일표본 Z (σ = 10,000) | −1.38185 | −1.381849 |
| 부호검정 증가 개수 | 120 | 120 |

모두 유효숫자 6자리까지 일치합니다.

# 6. 에이전트에 연결할 때 팁

Cortex Agent에 custom tool로 등록할 때는 **도구 설명(description)** 이 호출 정확도를 좌우합니다. 예시는 다음과 같습니다.

```text
1~2개 집단의 평균(또는 중앙값) 차이가 통계적으로 유의한지 검정한다.
t검정, Z검정, Mann-Whitney U, Wilcoxon 부호순위, 부호검정을 지원한다.
- 한 집단 평균을 특정 값과 비교: TEST_TYPE='one_sample', MU0에 기준값
- 서로 다른 두 집단 비교(A/B 테스트 등): TEST_TYPE='independent', GROUP_COL/GROUP_A/GROUP_B
- 같은 대상의 전후 비교: TEST_TYPE='paired', GROUP_COL=시점 컬럼, PAIR_ID_COL=대상 ID
- 모집단 표준편차를 알고 있다고 명시된 경우에만 z_one_sample / z_two_sample과 SIGMA 사용
- 사용자가 비모수 검정을 명시하면 mann_whitney / wilcoxon / sign 사용
  (wilcoxon, sign은 PAIR_ID_COL이 있으면 대응표본, 없으면 MU0 기준 일표본)
- 특정 기간이나 조건만 분석할 때는 FILTER_COL/FILTER_VALUE 사용
기본적으로 t검정을 호출하고, 결과의 recommended_test와 decision을 기준으로 답한다.
recommended_test가 비모수 검정이면 그 이유(소표본 + 정규성 위반)를 함께 설명한다.
세 집단 이상의 평균 비교에는 RUN_ANOVA를, 비율 비교에는 RUN_CHISQ를 사용한다.
```

함께 챙겨야 할 것들입니다.

- **기본은 t검정:** 사용자가 검정 방법을 지정하지 않으면 `one_sample`, `independent`, `paired` 중 하나를 쓰게 하세요. 비모수 대안이 함께 계산되고 필요하면 자동으로 권장되므로, 에이전트가 정규성을 미리 판단할 필요가 없습니다.
- **Z검정은 명시적인 경우에만:** "모표준편차가 10,000으로 알려져 있다"처럼 σ가 주어진 경우에만 Z검정을 쓰도록 안내하세요. 그렇지 않으면 t검정이 맞습니다.
- **독립표본과 대응표본의 혼동 주의:** 이 데이터는 같은 고객이 두 번 측정되었기 때문에, 전후 비교를 독립표본으로 잘못 돌리는 실수가 가능합니다. "같은 대상의 반복 측정이면 paired"라는 규칙을 명시하세요.
- **권한:** 에이전트를 사용하는 역할에 프로시저 USAGE 권한과 대상 테이블 SELECT 권한이 필요합니다.
- **대용량 데이터:** 프로시저가 데이터를 pandas로 읽어 들이므로, 큰 테이블은 `FILTER_COL`로 범위를 줄이거나 웨어하우스 크기를 고려해야 합니다.

# 7. 마무리

- 시맨틱 뷰만으로는 어려운 **t검정, Z검정, 비모수 검정과 p-value**를 Python 저장 프로시저 하나로 해결했습니다.
- **모수 / 비모수 자동 권장** 기능 덕분에, 에이전트가 정규성을 따로 판단하지 않아도 데이터에 맞는 검정으로 판정합니다.
- 함께 쓰는 도구:
  - **비율·범주 비교:** `RUN_CHISQ` (카이제곱, Fisher, 비율 Z, McNemar)
  - **세 집단 이상의 평균 비교:** `RUN_ANOVA` (분산분석, Welch ANOVA, Kruskal-Wallis, Friedman, 사후검정)
  - **전제조건 확인:** `RUN_ASSUMPTION` (정규성 검정, 등분산성 검정)

---

**환경:** Snowflake, Python 3.11 런타임, `scipy` 1.17.1 / `numpy` / `pandas` (Snowflake Anaconda 채널)
