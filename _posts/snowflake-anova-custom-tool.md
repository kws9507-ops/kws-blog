# Snowflake CoWork 에이전트를 위한 분산분석 커스텀 도구 만들기 (ANOVA · Welch · Kruskal-Wallis · Friedman · 사후검정)

> 세 집단 이상의 평균을 비교하는 **분산분석(ANOVA, F검정)** 과 그 대안인 **Welch ANOVA, Kruskal-Wallis, Friedman 검정**, 그리고 **사후검정(Tukey, Games-Howell, Dunn)** 을 Python 저장 프로시저 하나로 구현해, Snowflake CoWork 에이전트의 커스텀 도구로 쓸 수 있게 만드는 과정을 정리했습니다. [평균 비교 커스텀 도구](./snowflake-mean-test-custom-tool.md)의 3집단 이상 버전입니다.

## 목차
1. [왜 커스텀 도구가 필요한가](#1-왜-커스텀-도구가-필요한가)
2. [전체 구성](#2-전체-구성)
3. [샘플 데이터 만들기](#3-샘플-데이터-만들기)
4. [분산분석 프로시저 만들기](#4-분산분석-프로시저-만들기)
5. [실행 및 결과 해석](#5-실행-및-결과-해석)
6. [에이전트에 연결할 때 팁](#6-에이전트에-연결할-때-팁)
7. [마무리](#7-마무리)

---

## 1. 왜 커스텀 도구가 필요한가

Snowflake CoWork(구 Snowflake Intelligence)의 Cortex Analyst 도구는 자연어 질문을 **SQL로 바꿔 실행**합니다. 집단별 평균을 구하는 것은 쉽지만, 그 차이를 검정하는 일은 다릅니다.

| 항목 | SQL만으로 처리할 때 |
|---|---|
| 집단별 평균, 표준편차 | `GROUP BY`로 가능 |
| 제곱합(SS) 분해, F값 | 가능하지만 쿼리가 길어서 LLM이 매번 정확하게 생성하기 어려움 |
| p-value | F분포의 누적분포함수가 Snowflake SQL에 기본 제공되지 않음* |
| Welch ANOVA | 가중평균과 보정 자유도 공식이 복잡함 |
| 사후검정 | 스튜던트화 범위 분포(Tukey, Games-Howell)와 다중비교 보정이 필요해서 SQL로 구현하기 어려움 |
| 비모수 검정 | 순위 계산과 동점 보정이 필요함 |

\* 작성 시점 기준으로 확인한 내용입니다. 이후 Snowflake에 관련 함수가 추가되었을 수 있으니 공식 문서를 함께 확인하세요.

그래서 `scipy.stats`를 쓰는 **Python 저장 프로시저**를 만들고, 이를 에이전트의 custom tool로 등록하는 방식을 택했습니다.

### F검정과 분산분석의 관계

"F검정"이라는 이름은 두 가지 용도로 쓰입니다.

| 용도 | 묻는 질문 | F의 의미 | 이 시리즈의 도구 |
|---|---|---|---|
| **분산분석(ANOVA)** | 세 집단 이상의 **평균**이 같은가? | 집단 간 분산 / 집단 내 분산 | **`RUN_ANOVA`** (이 글) |
| 분산비 검정 | 두 집단의 **분산**이 같은가? | 분산 A / 분산 B | `RUN_ASSUMPTION` |

분산분석은 이름과 달리 **평균을 비교하는 검정**입니다. 전체 변동을 "집단 간 차이로 설명되는 부분"과 "집단 안의 무작위 변동"으로 나누고, 그 비율(F)이 충분히 크면 평균이 다르다고 판단합니다.

```
F = (집단 간 평균제곱 MS_between) / (집단 내 평균제곱 MS_within)
```

### 왜 t검정을 여러 번 하면 안 되나
집단이 3개면 쌍이 3개, 5개면 10개입니다. 유의수준 5%로 t검정을 10번 하면 **실제로 차이가 없어도 하나 이상 유의하게 나올 확률이 약 40%** 까지 올라갑니다(1 − 0.95¹⁰). 그래서 분산분석으로 "어딘가 차이가 있는지"를 먼저 한 번에 검정하고, 차이가 있으면 **다중비교 보정을 한 사후검정**으로 어느 쌍이 다른지 확인합니다.

### 이 도구가 다루는 질문: "3집단 이상의 평균(위치) 비교"

| 상황 | 모수 검정 | 등분산 위반 시 | 비모수 대안 | 사후검정 |
|---|---|---|---|---|
| 서로 다른 집단 3개 이상 | 일원분산분석 | Welch ANOVA | Kruskal-Wallis | Tukey HSD / Games-Howell / Dunn |
| 같은 대상을 3회 이상 측정 | (반복측정 ANOVA, 이번 범위에서 제외) | – | Friedman | Wilcoxon 쌍별 비교 (Holm 보정) |

## 2. 전체 구성

```
KWS_DB
└── STATISTICS (schema)
    ├── SAMPLE_ANOVA     -- 테스트용 샘플 테이블
    └── RUN_ANOVA(...)   -- 분산분석 Python 저장 프로시저 (VARIANT 반환)
```

동작 흐름은 다음과 같습니다.

```
사용자 질문 (CoWork)
  → 에이전트가 RUN_ANOVA 도구를 선택하고 파라미터를 채움
  → 프로시저가 데이터를 읽어 ANOVA, Welch, Kruskal-Wallis, 사후검정을 모두 계산
  → 정규성·등분산 결과에 따라 권장 검정과 권장 사후검정을 판단
  → JSON 결과 반환
  → 에이전트가 결과를 해석해서 답변
```

스키마는 다음과 같이 만듭니다.

```sql
CREATE SCHEMA IF NOT EXISTS KWS_DB.STATISTICS;
```

## 3. 샘플 데이터 만들기

### 3.1 시나리오

온라인 쇼핑몰 고객 300명의 **3개월간 월별 구매금액**을 분석하는 상황을 가정했습니다.

- 고객마다 **주 이용 채널**(APP / WEB / STORE)과 **거주 지역**(서울 / 부산 / 대구 / 광주)이 있습니다.
- 같은 고객의 **1월(M1), 2월(M2), 3월(M3)** 구매금액을 기록합니다.

이렇게 구성하면 하나의 테이블로 이 도구의 모든 검정을 테스트할 수 있습니다.

| 검정 | 이 데이터에서 묻는 질문 | 설계상 정답 |
|---|---|---|
| 일원분산분석: 채널별 (M1) | 채널에 따라 구매금액이 다른가? | **다름** (APP만 높음, STORE는 분산이 큼) |
| 일원분산분석: 지역별 (M1) | 지역에 따라 구매금액이 다른가? | **같음** |
| Friedman: 월별 (같은 고객) | 같은 고객의 구매금액이 월별로 달라지는가? | **달라짐** (매월 증가) |

### 3.2 테이블 구조

| 컬럼 | 타입 | 설명 |
|---|---|---|
| `CUSTOMER_ID` | NUMBER | 고객 ID (1~300). Friedman 검정에서 대상을 식별하는 키 |
| `CHANNEL` | VARCHAR | 주 이용 채널. `APP`, `WEB`, `STORE` (각 100명) |
| `REGION` | VARCHAR | 거주 지역. `SEOUL`, `BUSAN`, `DAEGU`, `GWANGJU` (무작위 배정) |
| `MONTH` | VARCHAR | 측정 월. `M1`, `M2`, `M3` |
| `PURCHASE_AMT` | NUMBER | 월 구매금액(원) |

고객 1명당 3개월치 3행이므로 **전체 900행**입니다. 채널과 지역은 고객 속성이라 같은 고객의 세 행에서 값이 같습니다.

### 3.3 데이터 생성 규칙

구매금액은 다음 네 가지 요소를 더해서 만들었습니다.

```
구매금액 = 채널 기본값 + 월 효과 + 고객 고유 효과 + 무작위 오차
```

| 요소 | 설정 | 의도 |
|---|---|---|
| 채널 기본값 | **APP 55,000**, WEB 50,000, STORE 50,000 | APP만 평균이 높도록 (사후검정에서 APP만 다르게) |
| 월 효과 | M1 +0, **M2 +1,500, M3 +3,000** | 매월 증가 추세 (Friedman 검정용) |
| 고객 고유 효과 | 정규분포, 평균 0 / 표준편차 6,000. 같은 고객의 세 달에 동일하게 적용 | 구매력이 큰 고객은 매달 큼 (반복측정 데이터의 특징) |
| 무작위 오차 | 정규분포, 표준편차 APP·WEB 5,000, **STORE 12,000** | STORE만 편차가 크도록 (등분산 위반 → Welch 권장) |
| 지역 | 구매금액에 영향 없음 | 지역별로는 차이가 없도록 |

고객 고유 효과와 오차를 합치면 한 달 기준 표준편차는 APP·WEB 약 7,800, STORE 약 13,400입니다.

### 3.4 생성 SQL

고객 속성(채널, 지역, 고유 효과)이 세 달 동안 같아야 하므로, 고객 테이블을 먼저 만들고 월과 교차 조인합니다.

```sql
-- 1) 고객 속성 (임시 테이블)
CREATE OR REPLACE TEMPORARY TABLE KWS_DB.STATISTICS.TMP_ANOVA_CUST AS
SELECT SEQ4() + 1 AS CUSTOMER_ID,
       DECODE(MOD(SEQ4(), 3), 0, 'APP', 1, 'WEB', 'STORE') AS CHANNEL,
       DECODE(UNIFORM(0, 3, RANDOM()), 0, 'SEOUL', 1, 'BUSAN', 2, 'DAEGU', 'GWANGJU') AS REGION,
       NORMAL(0, 1, RANDOM()) AS SUBJ_Z
FROM TABLE(GENERATOR(ROWCOUNT => 300));

-- 2) 고객 × 3개월
CREATE OR REPLACE TABLE KWS_DB.STATISTICS.SAMPLE_ANOVA AS
SELECT c.CUSTOMER_ID, c.CHANNEL, c.REGION, m.MONTH,
       ROUND(IFF(c.CHANNEL = 'APP', 55000, 50000)
             + m.MONTH_EFFECT
             + c.SUBJ_Z * 6000
             + NORMAL(0, 1, RANDOM()) * IFF(c.CHANNEL = 'STORE', 12000, 5000), 0) AS PURCHASE_AMT
FROM KWS_DB.STATISTICS.TMP_ANOVA_CUST c
CROSS JOIN (SELECT * FROM VALUES ('M1', 0), ('M2', 1500), ('M3', 3000) AS v(MONTH, MONTH_EFFECT)) m;
```

> **참고:** 시드 없이 `RANDOM()`을 쓰기 때문에 실행할 때마다 값이 달라집니다. 아래 수치는 필자의 실행 결과이며, 직접 실행하면 숫자는 조금씩 다르게 나옵니다. 특히 효과가 작은 쌍(예: STORE vs WEB)은 실행에 따라 유의 여부가 바뀔 수 있습니다.

### 3.5 생성된 데이터 요약 (필자 실행 기준)

**채널별 (M1)**

| 채널 | n | 평균 | 표준편차 |
|---|---|---|---|
| APP | 100 | 54,800 | 7,207 |
| STORE | 100 | 52,325 | **13,754** |
| WEB | 100 | 49,228 | 7,240 |

**지역별 (M1)**

| 지역 | n | 평균 | 표준편차 |
|---|---|---|---|
| BUSAN | 65 | 52,924 | 12,429 |
| DAEGU | 84 | 51,722 | 9,637 |
| GWANGJU | 74 | 52,186 | 9,552 |
| SEOUL | 77 | 51,803 | 9,104 |

**월별 (전체 고객)**

| 월 | n | 평균 | 표준편차 |
|---|---|---|---|
| M1 | 300 | 52,118 | 10,119 |
| M2 | 300 | 53,803 | 9,667 |
| M3 | 300 | 54,369 | 10,196 |

## 4. 분산분석 프로시저 만들기

### 4.1 설계 포인트

- **한 번 호출로 전부 계산:** `oneway`를 실행하면 일반 ANOVA, Welch ANOVA, Kruskal-Wallis, 세 가지 사후검정을 **모두** 계산합니다. 그리고 전제조건 결과를 보고 어느 결과를 쓸지(`recommended_test`, `recommended_posthoc`) 알려줍니다. 에이전트가 전제조건을 미리 판단하거나 도구를 여러 번 호출할 필요가 없습니다.
- **권장 검정 판정 규칙:**

| 조건 | 권장 검정 | 권장 사후검정 |
|---|---|---|
| 정규성 ○, 등분산 ○ | 일원분산분석 (ANOVA) | Tukey HSD |
| 정규성 ○, 등분산 ✕ | Welch ANOVA | Games-Howell |
| 정규성 ✕ | Kruskal-Wallis | Dunn (Holm 보정) |

  정규성은 집단별 Shapiro-Wilk로 판단하되, **집단 크기가 30 이상이면 중심극한정리에 따라 충족으로 간주**합니다. ANOVA는 표본이 크면 정규성 위반에 강건하기 때문입니다. 반면 등분산 위반은 표본이 커도 결과를 왜곡하므로 그대로 반영합니다.

- **Friedman 검정:** `TEST_TYPE='friedman'`이면 `SUBJECT_COL`로 같은 대상을 묶고, 모든 조건에 값이 있는 대상만 사용합니다. 제외된 대상 수는 `n_dropped_incomplete`로 알려줍니다. 사후검정으로는 조건 쌍별 Wilcoxon 부호순위 검정에 Holm 보정을 적용합니다.
- **효과크기:** η²(에타제곱), ω²(오메가제곱), Kruskal-Wallis의 ε², Friedman의 Kendall's W를 함께 반환합니다. ω²는 η²보다 편향이 적어서 표본이 작을 때 더 정확합니다.
- **scipy로 직접 구현한 부분:** Welch ANOVA, Games-Howell(scipy의 `studentized_range` 분포 사용), Dunn 검정(동점 보정 포함), Holm 보정은 별도 패키지 없이 공식대로 구현했습니다. 패키지를 추가하지 않으면 배포와 관리가 단순해집니다.
- **안전장치:**
  - 집단 수가 50개를 넘으면 오류를 반환합니다. 고객 ID 같은 컬럼을 집단으로 잘못 넣는 실수를 막기 위해서입니다.
  - `GROUPS`로 비교할 집단을 고를 수 있고, 없는 집단 값을 넣으면 그 값을 알려줍니다.
  - 집단 크기가 크게 불균형하거나(최대/최소 > 3) 집단이 2개뿐이면 경고를 반환합니다.
- **입력 검증과 오류 메시지:** 식별자를 정규식으로 검증하고, 쿼리는 Snowpark DataFrame API로 구성해서 SQL 인젝션을 막습니다. 필터 결과가 0행이거나 JSON 형식이 틀리면 원인을 알려줍니다.
- **정확한 수치 표현:** 결과는 유효숫자 6자리로 반올림합니다.

### 4.2 파라미터

| 파라미터 | 기본값 | 설명 |
|---|---|---|
| `TABLE_NAME` | (필수) | 분석할 테이블. `DB.SCHEMA.TABLE` 형식 |
| `VALUE_COL` | (필수) | 비교할 수치 컬럼 |
| `GROUP_COL` | (필수) | 집단 컬럼. Friedman에서는 조건(측정 시점) 컬럼 |
| `TEST_TYPE` | `oneway` | `oneway` / `friedman` |
| `GROUPS` | NULL | 비교할 집단 목록 JSON 배열, 예: `["APP","WEB"]`. 생략하면 전체 집단 |
| `SUBJECT_COL` | NULL | Friedman에서 같은 대상을 식별하는 키 (필수) |
| `ALPHA` | 0.05 | 유의수준 |
| `FILTER_COL`, `FILTER_VALUE` | NULL | 선택 사항. `FILTER_COL = FILTER_VALUE` 조건으로 데이터를 좁힘 |

### 4.3 반환 항목

**`oneway`**

| 구분 | 항목 |
|---|---|
| 기술통계 | 집단별 n, 평균, 표준편차, 중앙값, 최소, 최대 |
| ANOVA 표 | 집단 간·집단 내·전체의 제곱합(SS), 자유도, 평균제곱(MS), F, p-value |
| 효과크기 | η², ω², 해석 문구(`effect_size`: 거의 없음/작음/중간/큼) |
| 전제조건 | Levene(Brown-Forsythe) 등분산 검정, 집단별 Shapiro-Wilk, `normality_ok` |
| 대안 검정 | Welch ANOVA(F, df1, df2, p), Kruskal-Wallis(H, df, p, ε²) |
| 사후검정 | `posthoc_tukey`, `posthoc_games_howell`: 모든 쌍의 평균 차이, 신뢰구간, 보정 p, 유의 여부. `posthoc_dunn_holm`: z, p, Holm 보정 p, 유의 여부 |
| 판정 | `recommended_test`, `recommended_posthoc`, `warnings`, `decision` |

**`friedman`**

| 구분 | 항목 |
|---|---|
| 기본 정보 | 사용된 대상 수, 불완전해서 제외된 대상 수, 조건별 기술통계 |
| 검정 | Friedman χ², 자유도, p-value |
| 효과크기 | Kendall's W (0.1 작음, 0.3 중간, 0.5 큼) |
| 사후검정 | 조건 쌍별 중앙값 차이, Wilcoxon W, p, Holm 보정 p, 유의 여부 |

**효과크기 해석 기준 (η², ω², ε²)**

| 값 | 해석 |
|---|---|
| 0.01 미만 | 거의 없음 |
| 0.01 ~ 0.06 | 작음 |
| 0.06 ~ 0.14 | 중간 |
| 0.14 이상 | 큼 |

### 4.4 전체 코드

```sql
CREATE OR REPLACE PROCEDURE KWS_DB.STATISTICS.RUN_ANOVA(
    TABLE_NAME   VARCHAR,
    VALUE_COL    VARCHAR,
    GROUP_COL    VARCHAR,
    TEST_TYPE    VARCHAR DEFAULT 'oneway',
    GROUPS       VARCHAR DEFAULT NULL,
    SUBJECT_COL  VARCHAR DEFAULT NULL,
    ALPHA        FLOAT   DEFAULT 0.05,
    FILTER_COL   VARCHAR DEFAULT NULL,
    FILTER_VALUE VARCHAR DEFAULT NULL
)
RETURNS VARIANT
LANGUAGE PYTHON
RUNTIME_VERSION = '3.11'
PACKAGES = ('snowflake-snowpark-python', 'pandas', 'numpy', 'scipy')
HANDLER = 'main'
COMMENT = 'One-way ANOVA (F) with Welch, Kruskal-Wallis, Tukey HSD, Games-Howell, Dunn; Friedman repeated-measures test'
AS
$$
import itertools
import json
import math
import re

import numpy as np
import scipy
from scipy import stats
from snowflake.snowpark import Session
from snowflake.snowpark.functions import col

_IDENT = re.compile(r'^[A-Za-z_][A-Za-z0-9_$]*$')
MAX_GROUPS = 50


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


def _eta_label(e):
    if e is None:
        return None
    if e < 0.01:
        return '거의 없음'
    if e < 0.06:
        return '작음'
    if e < 0.14:
        return '중간'
    return '큼'


def _holm(pvals):
    m = len(pvals)
    order = np.argsort(pvals)
    adj = np.empty(m)
    running = 0.0
    for rank, idx in enumerate(order):
        running = max(running, min(1.0, (m - rank) * pvals[idx]))
        adj[idx] = running
    return adj


def _parse_groups(groups):
    if not groups:
        return None
    try:
        g = json.loads(groups)
    except Exception:
        raise ValueError('GROUPS must be a JSON array like ["A","B","C"]')
    if not isinstance(g, list) or not g:
        raise ValueError('GROUPS must be a JSON array like ["A","B","C"]')
    return [str(x) for x in g]


def _welch_anova(samples):
    k = len(samples)
    n = np.array([len(s) for s in samples], dtype=float)
    m = np.array([np.mean(s) for s in samples])
    v = np.array([np.var(s, ddof=1) for s in samples])
    if np.any(v == 0):
        return None
    w = n / v
    mw = np.sum(w * m) / np.sum(w)
    a = np.sum(w * (m - mw) ** 2) / (k - 1)
    tmp = np.sum((1 - w / np.sum(w)) ** 2 / (n - 1))
    b = 1 + 2 * (k - 2) / (k ** 2 - 1) * tmp
    f = a / b
    df2 = (k ** 2 - 1) / (3 * tmp)
    return {'F': _f(f), 'df1': k - 1, 'df2': _f(df2), 'p_value': _f(stats.f.sf(f, k - 1, df2))}


def _games_howell(samples, names, alpha):
    k = len(samples)
    out = []
    for i, j in itertools.combinations(range(k), 2):
        a, b = samples[i], samples[j]
        na, nb = len(a), len(b)
        va, vb = np.var(a, ddof=1) / na, np.var(b, ddof=1) / nb
        diff = np.mean(a) - np.mean(b)
        se = math.sqrt(va + vb)
        if se == 0:
            continue
        dfw = (va + vb) ** 2 / (va ** 2 / (na - 1) + vb ** 2 / (nb - 1))
        q = abs(diff) / se * math.sqrt(2)
        p = stats.studentized_range.sf(q, k, dfw)
        qcrit = stats.studentized_range.ppf(1 - alpha, k, dfw)
        half = qcrit * se / math.sqrt(2)
        out.append({'group_1': names[i], 'group_2': names[j], 'mean_diff': _f(diff),
                    'ci': [_f(diff - half), _f(diff + half)], 'df': _f(dfw),
                    'p_adj': _f(p), 'significant': bool(p < alpha)})
    return out


def _dunn(samples, names):
    allv = np.concatenate(samples)
    big_n = len(allv)
    ranks = stats.rankdata(allv)
    _, counts = np.unique(allv, return_counts=True)
    tie = np.sum(counts ** 3 - counts) / (12 * (big_n - 1))
    base = big_n * (big_n + 1) / 12 - tie
    mean_rank, pos = [], 0
    for s in samples:
        mean_rank.append(np.mean(ranks[pos:pos + len(s)]))
        pos += len(s)
    pairs, pv = [], []
    for i, j in itertools.combinations(range(len(samples)), 2):
        z = (mean_rank[i] - mean_rank[j]) / math.sqrt(base * (1 / len(samples[i]) + 1 / len(samples[j])))
        p = 2 * stats.norm.sf(abs(z))
        pairs.append((names[i], names[j], z, p))
        pv.append(p)
    adj = _holm(np.array(pv))
    return [{'group_1': a, 'group_2': b, 'z': _f(z), 'p_value': _f(p), 'p_holm': _f(pa)}
            for (a, b, z, p), pa in zip(pairs, adj)]


def main(session: Session, table_name, value_col, group_col, test_type, groups,
         subject_col, alpha, filter_col, filter_value):
    try:
        tbl = _check_ident(table_name, 'TABLE_NAME', parts=3)
        vcol = _check_ident(value_col, 'VALUE_COL')
        gcol = _check_ident(group_col, 'GROUP_COL')
        ttype = (test_type or 'oneway').strip().lower()
        alpha = 0.05 if alpha is None else float(alpha)
        if ttype not in ('oneway', 'friedman'):
            raise ValueError("TEST_TYPE must be one of: oneway, friedman")
        wanted = _parse_groups(groups)

        df = session.table(tbl)
        cols = set(df.columns)

        def need(name, label):
            c = _check_ident(name, label)
            if c not in cols:
                raise ValueError(f'column not found: {c}')
            return c

        for c in (vcol, gcol):
            if c not in cols:
                raise ValueError(f'column not found: {c}')
        if filter_col:
            fcol = need(filter_col, 'FILTER_COL')
            df = df.filter(col(fcol) == filter_value)

        result = {'test_type': ttype, 'table': tbl, 'value_col': vcol, 'group_col': gcol,
                  'alpha': alpha, 'scipy_version': scipy.__version__}
        if filter_col:
            result['filter'] = {filter_col.upper(): filter_value}

        sel = [gcol, vcol]
        if ttype == 'friedman':
            scol = need(subject_col, 'SUBJECT_COL')
            sel = [scol, gcol, vcol]
            result['subject_col'] = scol
        pdf = df.filter(col(gcol).is_not_null() & col(vcol).is_not_null()).select(*sel).to_pandas()
        if pdf.empty:
            raise ValueError('no rows match the given conditions (check FILTER_COL/FILTER_VALUE)')
        pdf[gcol] = pdf[gcol].astype(str)

        present = sorted(pdf[gcol].unique())
        if wanted:
            missing = [g for g in wanted if g not in present]
            if missing:
                raise ValueError(f'group values not found in {gcol}: {missing}')
            names = wanted
        else:
            names = present
        if len(names) > MAX_GROUPS:
            raise ValueError(f'{gcol} has {len(names)} groups (max {MAX_GROUPS}); use a categorical column or GROUPS')

        if ttype == 'friedman':
            if len(names) < 3:
                raise ValueError('friedman needs at least 3 conditions (use RUN_MEAN_TEST wilcoxon for 2)')
            wide = (pdf[pdf[gcol].isin(names)]
                    .pivot_table(index=scol, columns=gcol, values=vcol, aggfunc='mean')[names].dropna())
            n_sub = len(wide)
            if n_sub < 2:
                raise ValueError('need at least 2 subjects with values in every condition')
            data = [wide[c].to_numpy(dtype=float) for c in names]
            chi2, p = stats.friedmanchisquare(*data)
            k = len(names)
            kendall_w = chi2 / (n_sub * (k - 1))
            pairs, pv = [], []
            for i, j in itertools.combinations(range(k), 2):
                d = data[i] - data[j]
                if np.all(d == 0):
                    stat_, pp = 0.0, 1.0
                else:
                    stat_, pp = stats.wilcoxon(data[i], data[j])
                pairs.append((names[i], names[j], float(np.median(d)), stat_, pp))
                pv.append(pp)
            adj = _holm(np.array(pv))
            result.update({
                'n_subjects': int(n_sub),
                'n_dropped_incomplete': int(pdf[scol].nunique() - n_sub),
                'conditions': {c: _desc(x) for c, x in zip(names, data)},
                'friedman': {'chi2': _f(chi2), 'df': k - 1, 'p_value': _f(p)},
                'kendalls_w': _f(kendall_w),
                'posthoc_wilcoxon_holm': [
                    {'cond_1': a, 'cond_2': b, 'median_diff': _f(md), 'W': _f(s), 'p_value': _f(pp),
                     'p_holm': _f(pa), 'significant': bool(pa < alpha)}
                    for (a, b, md, s, pp), pa in zip(pairs, adj)],
                'decision': _decision(_f(p), alpha),
            })
            return result

        samples = [pdf.loc[pdf[gcol] == g, vcol].to_numpy(dtype=float) for g in names]
        if len(names) < 2:
            raise ValueError('need at least 2 groups')
        small = [g for g, s in zip(names, samples) if len(s) < 2]
        if small:
            raise ValueError(f'each group needs at least 2 values; too few in: {small}')

        k = len(names)
        allv = np.concatenate(samples)
        big_n = len(allv)
        grand = np.mean(allv)
        ss_b = float(sum(len(s) * (np.mean(s) - grand) ** 2 for s in samples))
        ss_w = float(sum(np.sum((s - np.mean(s)) ** 2) for s in samples))
        df_b, df_w = k - 1, big_n - k
        ms_b, ms_w = ss_b / df_b, ss_w / df_w
        f_val = ms_b / ms_w if ms_w > 0 else float('inf')
        p_anova = stats.f.sf(f_val, df_b, df_w)
        eta2 = ss_b / (ss_b + ss_w) if ss_b + ss_w > 0 else None
        omega2 = (ss_b - df_b * ms_w) / (ss_b + ss_w + ms_w) if ss_b + ss_w > 0 else None

        lev_w, lev_p = stats.levene(*samples, center='median')
        norm = {g: _normality(s) for g, s in zip(names, samples)}
        normal_ok = all(len(s) >= 30 or (norm[g] is None) or norm[g]['p_value'] >= alpha
                        for g, s in zip(names, samples))
        equal_var = lev_p >= alpha
        welch = _welch_anova(samples)
        h, p_kw = stats.kruskal(*samples)
        eps2 = (h - k + 1) / (big_n - k) if big_n > k else None

        tk = stats.tukey_hsd(*samples)
        ci = tk.confidence_interval(confidence_level=1 - alpha)
        tukey = []
        for i, j in itertools.combinations(range(k), 2):
            tukey.append({'group_1': names[i], 'group_2': names[j],
                          'mean_diff': _f(np.mean(samples[i]) - np.mean(samples[j])),
                          'ci': [_f(ci.low[i, j]), _f(ci.high[i, j])],
                          'p_adj': _f(tk.pvalue[i, j]),
                          'significant': bool(tk.pvalue[i, j] < alpha)})
        games_howell = _games_howell(samples, names, alpha)
        dunn = _dunn(samples, names)
        for d in dunn:
            d['significant'] = bool(d['p_holm'] is not None and d['p_holm'] < alpha)

        if not normal_ok:
            rec, p_rec, posthoc = 'kruskal_wallis', p_kw, 'dunn'
        elif not equal_var and welch is not None:
            rec, p_rec, posthoc = 'welch_anova', welch['p_value'], 'games_howell'
        else:
            rec, p_rec, posthoc = 'anova', p_anova, 'tukey'

        sizes = [len(s) for s in samples]
        warnings = []
        if max(sizes) / min(sizes) > 3:
            warnings.append('집단 크기 불균형이 큼 (최대/최소 > 3)')
        if k == 2:
            warnings.append('집단이 2개이면 RUN_MEAN_TEST(t검정)와 같은 결론 (F = t²)')

        result.update({
            'n': int(big_n), 'k': k,
            'groups': {g: _desc(s) for g, s in zip(names, samples)},
            'anova': {
                'between': {'SS': _f(ss_b), 'df': df_b, 'MS': _f(ms_b)},
                'within': {'SS': _f(ss_w), 'df': df_w, 'MS': _f(ms_w)},
                'total': {'SS': _f(ss_b + ss_w), 'df': big_n - 1},
                'F': _f(f_val), 'p_value': _f(p_anova)},
            'eta_squared': _f(eta2), 'omega_squared': _f(omega2), 'effect_size': _eta_label(eta2),
            'levene': {'W': _f(lev_w), 'p_value': _f(lev_p), 'equal_variance': bool(equal_var)},
            'normality': norm, 'normality_ok': bool(normal_ok),
            'welch_anova': welch,
            'kruskal_wallis': {'H': _f(h), 'df': k - 1, 'p_value': _f(p_kw), 'epsilon_squared': _f(eps2)},
            'posthoc_tukey': tukey,
            'posthoc_games_howell': games_howell,
            'posthoc_dunn_holm': dunn,
            'recommended_test': rec,
            'recommended_posthoc': posthoc,
            'warnings': warnings or None,
            'decision': _decision(_f(p_rec), alpha),
        })
        return result
    except Exception as e:
        return {'error': str(e)}
$$;
```

## 5. 실행 및 결과 해석

> 프로시저는 결과를 VARIANT로 반환합니다. 클라이언트에 따라 `[object Object]`처럼 보이면, 호출 직후 아래 쿼리로 JSON 문자열을 확인할 수 있습니다.
> ```sql
> SELECT TO_JSON($1) FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));
> ```

### 5.1 일원분산분석: 채널별 구매금액 (M1)

```sql
CALL KWS_DB.STATISTICS.RUN_ANOVA(
  TABLE_NAME => 'KWS_DB.STATISTICS.SAMPLE_ANOVA', VALUE_COL => 'PURCHASE_AMT',
  GROUP_COL => 'CHANNEL', FILTER_COL => 'MONTH', FILTER_VALUE => 'M1');
```

**ANOVA 표**

| 요인 | SS | df | MS | F | p |
|---|---|---|---|---|---|
| 집단 간 (채널) | 1.559 × 10⁹ | 2 | 7.796 × 10⁸ | 7.97 | 0.000426 |
| 집단 내 (오차) | 2.906 × 10¹⁰ | 297 | 9.784 × 10⁷ | | |
| 전체 | 3.062 × 10¹⁰ | 299 | | | |

**전제조건과 대안 검정**

| 항목 | 값 |
|---|---|
| η² / ω² | 0.051 / 0.044 (작음) |
| Levene p | 1.96 × 10⁻¹² → **등분산 가정 위반** |
| 정규성 | 집단별 n = 100 ≥ 30 → 충족으로 간주 |
| **Welch ANOVA** | **F = 14.85, df = (2, 188.5), p = 1.03 × 10⁻⁶** |
| Kruskal-Wallis | H = 20.16, p = 4.19 × 10⁻⁵, ε² = 0.061 |
| 권장 검정 / 사후검정 | **welch_anova / games_howell** |
| 판정 | **유의함** |

**사후검정 비교**

| 쌍 | 평균 차이 | Tukey (등분산 가정) | **Games-Howell (권장)** | Dunn (순위 기반) |
|---|---|---|---|---|
| APP − WEB | +5,573 | p = 0.00025 ✔ | **p = 4.3 × 10⁻⁷ ✔** (CI: 3,160 ~ 7,985) | p = 2.2 × 10⁻⁵ ✔ |
| APP − STORE | +2,475 | p = 0.182 | **p = 0.251** (CI: −1,201 ~ 6,151) | p = 0.045 ✔ |
| STORE − WEB | +3,098 | p = 0.070 | **p = 0.117** (CI: −582 ~ 6,777) | p = 0.027 ✔ |

**해석:** 채널에 따라 구매금액이 다르고, 그 차이는 주로 **APP과 WEB 사이**에서 납니다(약 5,600원). STORE는 분산이 커서 다른 채널과의 평균 차이가 확실하지 않습니다. 등분산 가정이 깨졌으므로 Welch ANOVA와 Games-Howell 결과를 기준으로 판단합니다.

> **세 사후검정의 결과가 다른 이유:**
> - **Tukey**는 모든 집단의 분산이 같다고 보고 합동분산을 씁니다. 그래서 STORE의 큰 분산이 다른 쌍의 비교에도 섞여 들어갑니다.
> - **Games-Howell**은 쌍마다 각자의 분산을 쓰므로, STORE가 포함된 쌍은 불확실성이 크게 반영되어 보수적으로 나옵니다.
> - **Dunn**은 값 대신 순위를 비교합니다. STORE는 분산은 크지만 중앙값이 WEB보다 높아서, 순위 기준으로는 차이가 잡힙니다.
>
> 같은 데이터라도 **무엇을 비교하느냐(평균인지 순위 분포인지)와 어떤 가정을 하느냐**에 따라 결론이 달라질 수 있습니다. 그래서 도구가 전제조건에 맞는 검정을 `recommended_posthoc`으로 알려줍니다.

### 5.2 일원분산분석: 지역별 구매금액 (M1)

```sql
CALL KWS_DB.STATISTICS.RUN_ANOVA(
  TABLE_NAME => 'KWS_DB.STATISTICS.SAMPLE_ANOVA', VALUE_COL => 'PURCHASE_AMT',
  GROUP_COL => 'REGION', FILTER_COL => 'MONTH', FILTER_VALUE => 'M1');
```

| 항목 | 값 |
|---|---|
| F / p | 0.20 / 0.893 |
| η² | 0.002 (거의 없음) |
| Levene p | 0.259 → 등분산 가정 만족 |
| Welch ANOVA p | 0.922 |
| Kruskal-Wallis p | 0.910 |
| 권장 검정 | anova |
| 판정 | **유의하지 않음** |

**해석:** 지역에 따라 구매금액이 다르다고 볼 근거가 없습니다. 세 검정이 모두 같은 결론입니다. 전체 검정이 유의하지 않으므로 사후검정 결과는 해석하지 않습니다.

### 5.3 Friedman 검정: 같은 고객의 월별 구매금액

```sql
CALL KWS_DB.STATISTICS.RUN_ANOVA(
  TABLE_NAME => 'KWS_DB.STATISTICS.SAMPLE_ANOVA', VALUE_COL => 'PURCHASE_AMT',
  GROUP_COL => 'MONTH', TEST_TYPE => 'friedman', SUBJECT_COL => 'CUSTOMER_ID');
```

| 항목 | 값 |
|---|---|
| 사용된 고객 수 | 300 (제외 0) |
| Friedman χ² / df / p | 22.17 / 2 / 1.54 × 10⁻⁵ |
| Kendall's W | 0.037 (작음) |
| 판정 | **유의함** |

**사후검정 (Wilcoxon, Holm 보정)**

| 쌍 | 중앙값 차이 | 보정 p | 유의 |
|---|---|---|---|
| M1 − M2 | −1,943 | 0.00136 | ✔ |
| M1 − M3 | −3,061 | 3.08 × 10⁻⁶ | ✔ |
| M2 − M3 | −1,412 | 0.0787 | |

**해석:** 같은 고객의 구매금액이 월별로 달라지며, 1월보다 2월과 3월이 유의하게 높습니다. 2월과 3월의 차이는 유의하지 않습니다. 설계상 월 효과는 매달 +1,500원이었으므로 M2−M3도 실제로는 차이가 있지만, 이 표본 크기에서는 그 차이를 검출하지 못했습니다.

> **왜 일원분산분석이 아니라 Friedman인가:** 같은 고객이 세 번 측정되었기 때문에 세 달의 값은 서로 독립이 아닙니다. 구매력이 큰 고객은 매달 큽니다. Friedman 검정은 **각 고객 안에서 월별 순위**를 비교하므로 이 고객 간 차이를 제거하고 월 효과만 봅니다. 고객 간 편차(표준편차 6,000)가 월 효과(1,500)보다 훨씬 크기 때문에, 일반 분산분석으로는 월 효과가 묻히기 쉽습니다.

### 5.4 비모수 검정으로 자동 전환되는 경우

이상치가 섞인 소표본(집단당 10개)으로 자동 전환 기능을 확인했습니다.

```
A: 1.0, 1.2, 1.1, 1.3, 1.0, 1.4, 1.2, 1.1, 1.0, 25.0
B: 3.0, 3.5, 2.8, 3.1, 3.3, 2.9, 3.2, 3.4, 3.0, 40.0
C: 5.0, 5.2, 4.9, 5.1, 5.3, 5.0, 4.8, 5.4, 5.1, 60.0
```

| 항목 | 결과 |
|---|---|
| 정규성 (`normality_ok`) | false (n < 30이고 Shapiro-Wilk 기각) |
| 일원분산분석 p | 0.480 → 유의하지 않음 |
| Kruskal-Wallis p | 0.000168 → 유의함 |
| `recommended_test` / `recommended_posthoc` | **kruskal_wallis / dunn** |
| Dunn (Holm 보정) | A−C만 유의 (p = 9.1 × 10⁻⁵), A−B·B−C는 p = 0.074 |
| `decision` | **유의함** |

**해석:** 값 대부분은 A < B < C로 뚜렷하게 다른데, 극단값 때문에 집단 내 분산이 커져서 분산분석이 차이를 잡아내지 못했습니다. 도구가 정규성 위반을 감지하고 Kruskal-Wallis로 판정을 바꿨습니다.

### 5.5 집단 선택과 오류 처리

**`GROUPS`로 일부 집단만 비교:**
```sql
CALL KWS_DB.STATISTICS.RUN_ANOVA(
  TABLE_NAME => 'KWS_DB.STATISTICS.SAMPLE_ANOVA', VALUE_COL => 'PURCHASE_AMT',
  GROUP_COL => 'CHANNEL', GROUPS => '["WEB","STORE"]',
  FILTER_COL => 'MONTH', FILTER_VALUE => 'M1');
```
집단이 2개가 되므로 결과와 함께 "집단이 2개이면 RUN_MEAN_TEST(t검정)와 같은 결론 (F = t²)"이라는 경고가 반환됩니다.

| 입력 | 반환되는 오류 |
|---|---|
| `GROUP_COL => 'CUSTOMER_ID'` (집단 300개) | `CUSTOMER_ID has 300 groups (max 50); use a categorical column or GROUPS` |
| `GROUPS => '["APP","XXX"]'` | `group values not found in CHANNEL: ['XXX']` |
| `GROUPS => 'APP,WEB'` (JSON 아님) | `GROUPS must be a JSON array like ["A","B","C"]` |
| 필터 결과 0행 | `no rows match the given conditions (check FILTER_COL/FILTER_VALUE)` |
| Friedman인데 `SUBJECT_COL` 누락 | `SUBJECT_COL is required` |
| 컬럼명에 `; DROP TABLE X` | `invalid GROUP_COL: ...` (쿼리 실행 안 됨) |

### 5.6 SQL로 직접 계산한 값과 교차검증

| 지표 | 프로시저 | SQL 직접 계산 |
|---|---|---|
| SS (집단 간) | 1.55924 × 10⁹ | 1,559,237,880 |
| SS (집단 내) | 2.90589 × 10¹⁰ | 29,058,890,789 |
| F | 7.96819 | 7.968192 |
| η² | 0.0509253 | 0.0509253 |
| Welch F / df2 | 14.8475 / 188.458 | 14.847458 / 188.458184 |
| Friedman χ² | 22.1667 | 22.166667 |

모두 유효숫자 6자리까지 일치합니다.

## 6. 에이전트에 연결할 때 팁

Cortex Agent에 custom tool로 등록할 때는 **도구 설명(description)** 이 호출 정확도를 좌우합니다. 예시는 다음과 같습니다.

```text
세 개 이상 집단의 평균 차이를 검정한다 (분산분석, ANOVA, F검정).
일원분산분석, Welch ANOVA, Kruskal-Wallis, Friedman 검정과
사후검정(Tukey HSD, Games-Howell, Dunn)을 지원한다.
- 서로 다른 집단 3개 이상의 평균 비교 (예: 채널별, 지역별, 등급별 구매금액):
  TEST_TYPE='oneway', GROUP_COL=집단 컬럼
- 같은 대상을 3회 이상 측정한 값의 비교 (예: 같은 고객의 월별 구매금액):
  TEST_TYPE='friedman', GROUP_COL=측정 시점 컬럼, SUBJECT_COL=대상 ID
- 일부 집단만 비교할 때는 GROUPS에 JSON 배열, 특정 기간만 분석할 때는 FILTER_COL/FILTER_VALUE
결과의 recommended_test와 decision으로 전체 차이를 판단하고,
유의하면 recommended_posthoc에 해당하는 사후검정 결과로 어느 집단끼리 다른지 설명한다.
유의하지 않으면 사후검정 결과는 해석하지 않는다.
효과크기는 eta_squared와 effect_size로 설명한다.
두 집단 비교에는 RUN_MEAN_TEST를, 집단 간 분산 비교에는 RUN_ASSUMPTION을 사용한다.
```

함께 챙겨야 할 것들입니다.

- **사후검정은 권장 결과만:** 세 가지 사후검정 결과가 모두 반환되므로, 에이전트가 유리한 결과를 골라 쓰지 않도록 `recommended_posthoc` 기준으로만 답하게 하세요.
- **전체 검정이 먼저:** 전체 검정이 유의하지 않은데 사후검정에서 일부 쌍이 유의하게 나오는 경우가 있습니다. 이때는 "차이가 있다"고 결론 내리지 않도록 지침에 명시하세요.
- **반복측정 데이터 주의:** 같은 대상을 여러 번 측정한 데이터에 `oneway`를 쓰면 독립성 가정이 깨집니다. "같은 고객", "같은 매장의 월별"처럼 반복측정이면 `friedman`을 쓰도록 안내하세요.
- **"F검정"이라는 표현:** 사용자가 "F검정"이라고만 하면 평균 비교(이 도구)인지 분산 비교(`RUN_ASSUMPTION`)인지 모호합니다. 질문의 대상이 평균인지 분산(편차)인지로 구분하게 하세요.
- **권한:** 에이전트를 사용하는 역할에 프로시저 USAGE 권한과 대상 테이블 SELECT 권한이 필요합니다.
- **대용량 데이터:** 사후검정과 비모수 검정에 원본 값이 필요해서 데이터를 pandas로 읽어 들입니다. 큰 테이블은 `FILTER_COL`로 범위를 줄이거나 웨어하우스 크기를 고려해야 합니다.

## 7. 마무리

- 시맨틱 뷰만으로는 어려운 **분산분석(F검정), Welch ANOVA, 비모수 검정, 사후검정**을 Python 저장 프로시저 하나로 해결했습니다.
- **한 번 호출로 모든 대안을 계산**하고 전제조건에 맞는 검정과 사후검정을 권장하므로, 에이전트가 통계적 판단을 대신 내릴 필요가 없습니다.
- 추가 패키지 없이 scipy만으로 Welch ANOVA, Games-Howell, Dunn 검정을 구현했습니다.
- 함께 쓰는 도구:
  - **1~2집단 평균 비교:** `RUN_MEAN_TEST` (t검정, Z검정, Mann-Whitney, Wilcoxon, 부호검정)
  - **비율·범주 비교:** `RUN_CHISQ` (카이제곱, Fisher, 비율 Z, McNemar)
  - **전제조건 확인:** `RUN_ASSUMPTION` (정규성 검정, 등분산성 검정, 분산비 F검정)

---

**환경:** Snowflake, Python 3.11 런타임, `scipy` 1.17.1 / `numpy` / `pandas` (Snowflake Anaconda 채널)
