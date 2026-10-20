# Snowflake CoWork 에이전트를 위한 카이제곱 검정 커스텀 도구 만들기

> 범주형 데이터의 관련성과 분포를 검정하는 **카이제곱 검정**을 Python 저장 프로시저로 구현해, Snowflake CoWork 에이전트의 커스텀 도구로 쓸 수 있게 만드는 과정을 정리했습니다. 앞선 글 [t검정 커스텀 도구 만들기](./snowflake-ttest-custom-tool.md)와 같은 구조로 만들었습니다.

## 목차
1. [왜 커스텀 도구가 필요한가](#1-왜-커스텀-도구가-필요한가)
2. [전체 구성](#2-전체-구성)
3. [샘플 데이터 만들기](#3-샘플-데이터-만들기)
4. [카이제곱 검정 프로시저 만들기](#4-카이제곱-검정-프로시저-만들기)
5. [실행 및 결과 해석](#5-실행-및-결과-해석)
6. [에이전트에 연결할 때 팁](#6-에이전트에-연결할-때-팁)
7. [마무리](#7-마무리)

---

## 1. 왜 커스텀 도구가 필요한가

Snowflake CoWork(구 Snowflake Intelligence)의 Cortex Analyst 도구는 자연어 질문을 **SQL로 바꿔 실행**합니다. 교차표 작성까지는 SQL만으로 충분하지만, 카이제곱 검정은 그렇지 않습니다.

| 항목 | SQL만으로 처리할 때 |
|---|---|
| 교차표, 행·열 비율 | `GROUP BY`, `PIVOT`, `COUNT / SUM() OVER()`로 가능 |
| 기대빈도, χ² 통계량 | 계산은 가능하지만 쿼리가 길어서 LLM이 매번 정확하게 생성하기 어려움 |
| p-value | 카이제곱 분포의 누적분포함수가 Snowflake SQL에 기본 제공되지 않음* |
| Fisher 정확검정 | SQL로 구현하기 어려움 |
| Cramér's V, 조정잔차 | 가능하지만 쿼리가 더 복잡해짐 |

\* 작성 시점 기준으로 확인한 내용입니다. 이후 Snowflake에 관련 함수가 추가되었을 수 있으니 공식 문서를 함께 확인하세요.

그래서 `scipy.stats`를 쓰는 **Python 저장 프로시저**를 만들고, 이를 에이전트의 custom tool로 등록하는 방식을 택했습니다.

### 이 도구로 할 수 있는 검정

| 검정 | 질문 형태 | 예시 |
|---|---|---|
| 독립성 검정 | 두 범주형 변수가 서로 관련 있는가? | 실험 그룹과 구매 여부는 관련이 있는가? |
| 동질성 검정 | 여러 집단의 범주 분포가 같은가? | A/B 그룹의 채널 이용 분포가 같은가? (계산은 독립성 검정과 같음) |
| 적합도 검정 | 관측 분포가 기대 비율과 같은가? | 방문 요일이 균등하게 분포하는가? |
| Fisher 정확검정 | 2×2 표에서 기대빈도가 작을 때 | 표본이 작은 A/B 테스트 |

## 2. 전체 구성

```
KWS_DB
└── STATISTICS (schema)
    ├── SAMPLE_CHISQ     -- 테스트용 범주형 샘플 테이블
    └── RUN_CHISQ(...)   -- 카이제곱 검정 Python 저장 프로시저 (VARIANT 반환)
```

동작 흐름은 다음과 같습니다.

```
사용자 질문 (CoWork)
  → 에이전트가 RUN_CHISQ 도구를 선택하고 파라미터를 채움
  → Snowflake에서 빈도를 집계 (GROUP BY)
  → 집계된 빈도표만 Python으로 가져와 scipy로 검정
  → JSON 결과 반환
  → 에이전트가 결과를 해석해서 답변
```

스키마는 다음과 같이 만듭니다.

```sql
CREATE SCHEMA IF NOT EXISTS KWS_DB.STATISTICS;
```

## 3. 샘플 데이터 만들기

### 3.1 시나리오

t검정 글과 같은 **온라인 쇼핑몰 프로모션 실험**을 범주형 관점에서 다시 봅니다.

- 고객 600명을 무작위로 **A그룹(기존안)** 과 **B그룹(신규 프로모션)** 으로 나눕니다.
- 고객별로 **구매 여부**, **주로 방문한 요일**, **주 이용 채널**을 기록합니다.

이렇게 구성하면 하나의 테이블로 "관련 있음"과 "관련 없음"을 모두 테스트할 수 있습니다.

| 검정 | 이 데이터에서 묻는 질문 | 설계상 정답 |
|---|---|---|
| 독립성: 그룹 × 구매여부 | 프로모션 그룹에 따라 구매율이 다른가? | **관련 있음** |
| 독립성: 그룹 × 채널 | 그룹에 따라 이용 채널 분포가 다른가? | **관련 없음** |
| 적합도: 방문요일 (균등 기준) | 요일별 방문이 균등한가? | **균등하지 않음** |
| 적합도: 방문요일 (설계 비율 기준) | 요일별 방문이 설계한 비율을 따르는가? | **따름** |

### 3.2 테이블 구조

| 컬럼 | 타입 | 설명 |
|---|---|---|
| `CUSTOMER_ID` | NUMBER | 고객 ID (1~600) |
| `AB_GROUP` | VARCHAR | 실험 그룹. `A`(기존안) 또는 `B`(신규 프로모션), 약 50:50 무작위 배정 |
| `PURCHASED` | VARCHAR | 구매 여부. `Y` 또는 `N` |
| `VISIT_DAY` | VARCHAR | 주 방문 요일. `MON` ~ `SUN` |
| `CHANNEL` | VARCHAR | 주 이용 채널. `APP`, `WEB`, `STORE` |

고객 1명당 1행이므로 **전체 600행**입니다.

### 3.3 데이터 생성 규칙

| 컬럼 | 생성 규칙 | 의도 |
|---|---|---|
| `AB_GROUP` | A/B를 각각 50% 확률로 배정 | 무작위 배정 |
| `PURCHASED` | 구매확률 **A 18%, B 30%** | 그룹과 **관련 있게** 설계 |
| `VISIT_DAY` | 월~목 각 12%, 금 14%, **토·일 각 19%** | 주말에 몰리도록 **균등하지 않게** 설계 |
| `CHANNEL` | APP/WEB/STORE를 각각 1/3 확률로 배정, 그룹과 무관 | 그룹과 **관련 없게** 설계 |

### 3.4 생성 SQL

```sql
CREATE OR REPLACE TABLE KWS_DB.STATISTICS.SAMPLE_CHISQ AS
WITH c AS (
  SELECT SEQ4() + 1 AS CUSTOMER_ID,
         IFF(UNIFORM(0, 1, RANDOM()) = 0, 'A', 'B') AS AB_GROUP,
         UNIFORM(0, 99, RANDOM()) AS R_BUY,
         UNIFORM(0, 99, RANDOM()) AS R_DAY,
         UNIFORM(0, 2, RANDOM()) AS R_CH
  FROM TABLE(GENERATOR(ROWCOUNT => 600))
)
SELECT CUSTOMER_ID,
       AB_GROUP,
       IFF(R_BUY < IFF(AB_GROUP = 'B', 30, 18), 'Y', 'N') AS PURCHASED,
       CASE WHEN R_DAY < 12 THEN 'MON' WHEN R_DAY < 24 THEN 'TUE' WHEN R_DAY < 36 THEN 'WED'
            WHEN R_DAY < 48 THEN 'THU' WHEN R_DAY < 62 THEN 'FRI' WHEN R_DAY < 81 THEN 'SAT'
            ELSE 'SUN' END AS VISIT_DAY,
       DECODE(R_CH, 0, 'APP', 1, 'WEB', 'STORE') AS CHANNEL
FROM c;
```

`UNIFORM(0, 99, RANDOM())`으로 0~99 사이 정수를 뽑아, 그 값이 어느 구간에 들어가는지로 확률을 표현했습니다. 예를 들어 B그룹은 `R_BUY < 30`이면 구매(Y)이므로 구매확률이 30%입니다.

> **참고:** 시드 없이 `RANDOM()`을 쓰기 때문에 실행할 때마다 값이 달라집니다. 아래 수치는 필자의 실행 결과이며, 직접 실행하면 숫자는 조금씩 다르게 나옵니다. 다만 설계한 차이가 충분히 커서 결론은 대체로 같게 나옵니다.

### 3.5 생성된 데이터 요약 (필자 실행 기준)

**그룹 × 구매여부**

| 그룹 | N | Y | 합계 | 구매율 |
|---|---|---|---|---|
| A | 253 | 51 | 304 | 16.8% |
| B | 194 | 102 | 296 | 34.5% |

**그룹 × 채널**

| 그룹 | APP | STORE | WEB | 합계 |
|---|---|---|---|---|
| A | 101 | 100 | 103 | 304 |
| B | 92 | 107 | 97 | 296 |

**방문요일**

| MON | TUE | WED | THU | FRI | SAT | SUN | 합계 |
|---|---|---|---|---|---|---|---|
| 65 | 72 | 62 | 70 | 88 | 123 | 120 | 600 |

## 4. 카이제곱 검정 프로시저 만들기

### 4.1 설계 포인트

- **두 가지 검정을 하나의 도구로:** `TEST_TYPE` 파라미터로 `independence`(독립성·동질성)와 `goodness_of_fit`(적합도)을 선택합니다.
- **집계는 Snowflake에서:** 원본 행을 모두 가져오지 않고 `GROUP BY`로 빈도를 집계한 뒤 빈도표만 Python으로 가져옵니다. 그래서 수억 건 테이블에서도 메모리 부담이 거의 없습니다. t검정 도구와 다른 점입니다.
- **전제조건 자동 점검:** 기대빈도 5 미만 셀의 비율을 계산합니다. 2×2 표에서 이런 셀이 있으면 Fisher 정확검정을 권장 검정으로 판정하고, 그보다 큰 표에서 20%를 넘으면 경고를 반환합니다.
- **해석에 필요한 정보:** p-value뿐 아니라 연관 강도(Cramér's V)와 **어느 셀이 차이를 만드는지**(조정잔차)를 함께 반환해서, LLM이 "B그룹의 구매가 기대보다 많다"처럼 구체적으로 설명할 수 있게 합니다.
- **입력 검증:** 테이블명과 컬럼명을 정규식으로 검증하고, 실제 테이블에 있는 컬럼인지 확인합니다. 쿼리는 Snowpark DataFrame API로 구성해서 SQL 인젝션을 막습니다.
- **정확한 수치 표현:** 결과는 유효숫자 6자리로 반올림합니다. 소수 자릿수로 반올림하면 `6.75e-7` 같은 아주 작은 p-value가 모두 `0.000001`로 뭉개지기 때문입니다.

### 4.2 파라미터

| 파라미터 | 기본값 | 설명 |
|---|---|---|
| `TABLE_NAME` | (필수) | 분석할 테이블. `DB.SCHEMA.TABLE` 형식 |
| `ROW_COL` | (필수) | 첫 번째 범주형 컬럼. 적합도 검정에서는 검정 대상 컬럼 |
| `COL_COL` | NULL | 두 번째 범주형 컬럼 (독립성 검정에서 필수) |
| `TEST_TYPE` | `independence` | `independence` / `goodness_of_fit` |
| `EXPECTED_PROBS` | NULL | 적합도 검정의 기대 비율. JSON 객체, 예: `{"A":0.5,"B":0.5}`. 합이 1이 아니어도 자동으로 정규화. 생략하면 균등분포 |
| `ALPHA` | 0.05 | 유의수준 |
| `FILTER_COL`, `FILTER_VALUE` | NULL | 선택 사항. `FILTER_COL = FILTER_VALUE` 조건으로 데이터를 좁힘 |

### 4.3 반환 항목

| 검정 | 반환 항목 |
|---|---|
| 공통 | 검정 유형, 테이블, 컬럼, 유의수준, 판정 문구(`decision`) |
| 독립성 | 표본 수, 행·열 범주, 셀별 관측빈도·기대빈도·행 %·조정잔차, Pearson χ²·자유도·p-value, Cramér's V와 연관 강도, 기대빈도 5 미만 셀 비율, 권장 검정 |
| 독립성 (2×2) | 위 항목에 더해 Yates 보정 결과, Fisher 정확검정(오즈비, p-value) |
| 적합도 | 기대 비율 기준(균등/사용자 지정), 범주별 관측빈도·기대빈도·관측 %·표준화잔차, χ²·자유도·p-value, 기대빈도 5 미만 범주 수 |

**결과 읽는 법**

| 지표 | 해석 기준 |
|---|---|
| Cramér's V | 0.1 미만 거의 없음, 0.1~0.3 약함, 0.3~0.5 중간, 0.5 이상 강함 (도구가 `association_strength`로 함께 반환) |
| 조정잔차 / 표준화잔차 | 절댓값이 약 2(정확히는 1.96)를 넘으면 그 셀이 기대보다 유의하게 많거나(+) 적음(−) |
| 오즈비 (2×2) | 첫 번째 행의 오즈를 두 번째 행의 오즈로 나눈 값. 1이면 관련 없음 |

### 4.4 전체 코드

```sql
CREATE OR REPLACE PROCEDURE KWS_DB.STATISTICS.RUN_CHISQ(
    TABLE_NAME     VARCHAR,
    ROW_COL        VARCHAR,
    COL_COL        VARCHAR DEFAULT NULL,
    TEST_TYPE      VARCHAR DEFAULT 'independence',
    EXPECTED_PROBS VARCHAR DEFAULT NULL,
    ALPHA          FLOAT   DEFAULT 0.05,
    FILTER_COL     VARCHAR DEFAULT NULL,
    FILTER_VALUE   VARCHAR DEFAULT NULL
)
RETURNS VARIANT
LANGUAGE PYTHON
RUNTIME_VERSION = '3.11'
PACKAGES = ('snowflake-snowpark-python', 'pandas', 'numpy', 'scipy')
HANDLER = 'main'
COMMENT = 'Chi-square test of independence / goodness of fit with Cramer V, residuals, Fisher exact for sparse 2x2'
AS
$$
import json
import math
import re

import numpy as np
from scipy import stats
from snowflake.snowpark import Session
from snowflake.snowpark.functions import col, count, lit

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


def _decision(p, alpha):
    if p is None:
        return None
    return '유의함 (귀무가설 기각)' if p < alpha else '유의하지 않음 (귀무가설 채택)'


def _strength(v):
    if v is None:
        return None
    if v < 0.1:
        return '거의 없음'
    if v < 0.3:
        return '약함'
    if v < 0.5:
        return '중간'
    return '강함'


def main(session: Session, table_name, row_col, col_col, test_type, expected_probs,
         alpha, filter_col, filter_value):
    try:
        tbl = _check_ident(table_name, 'TABLE_NAME', parts=3)
        rcol = _check_ident(row_col, 'ROW_COL')
        ttype = (test_type or 'independence').strip().lower()
        alpha = 0.05 if alpha is None else float(alpha)
        if ttype not in ('independence', 'goodness_of_fit'):
            raise ValueError("TEST_TYPE must be one of: independence, goodness_of_fit")

        df = session.table(tbl)
        cols = set(df.columns)

        def need(name, label):
            c = _check_ident(name, label)
            if c not in cols:
                raise ValueError(f'column not found: {c}')
            return c

        need(rcol, 'ROW_COL')
        if filter_col:
            fcol = need(filter_col, 'FILTER_COL')
            df = df.filter(col(fcol) == filter_value)

        result = {'test_type': ttype, 'table': tbl, 'row_col': rcol, 'alpha': alpha}
        if filter_col:
            result['filter'] = {filter_col.upper(): filter_value}

        if ttype == 'goodness_of_fit':
            agg = (df.filter(col(rcol).is_not_null()).group_by(rcol)
                     .agg(count(lit(1)).alias('N')).to_pandas())
            observed = {str(k): int(v) for k, v in zip(agg[rcol], agg['N'])}
            if expected_probs:
                probs = json.loads(expected_probs)
                if not isinstance(probs, dict) or not probs:
                    raise ValueError('EXPECTED_PROBS must be a JSON object like {"A": 0.5, "B": 0.5}')
                probs = {str(k): float(v) for k, v in probs.items()}
                extra = sorted(set(observed) - set(probs))
                if extra:
                    raise ValueError(f'categories missing in EXPECTED_PROBS: {extra}')
                if any(v < 0 for v in probs.values()) or sum(probs.values()) <= 0:
                    raise ValueError('EXPECTED_PROBS must be non-negative with positive sum')
                expected_basis = 'user_specified'
            else:
                probs = {k: 1.0 for k in observed}
                expected_basis = 'uniform'
            cats = sorted(probs)
            total_p = sum(probs.values())
            obs = np.array([observed.get(c, 0) for c in cats], dtype=float)
            n = obs.sum()
            if n == 0 or len(cats) < 2:
                raise ValueError('need at least 2 categories with data')
            exp = np.array([probs[c] / total_p * n for c in cats])
            chi2, p = stats.chisquare(obs, exp)
            low = int((exp < 5).sum())
            result.update({
                'expected_basis': expected_basis,
                'n': int(n),
                'categories': {
                    c: {'observed': int(o), 'expected': _f(e),
                        'observed_pct': _f(o / n * 100),
                        'std_residual': _f((o - e) / math.sqrt(e)) if e > 0 else None}
                    for c, o, e in zip(cats, obs, exp)
                },
                'chi2': _f(chi2), 'df': len(cats) - 1, 'p_value': _f(p),
                'expected_lt5_cells': low,
                'warning': '기대빈도 5 미만 범주가 있어 결과 신뢰도가 낮을 수 있음' if low else None,
                'decision': _decision(_f(p), alpha),
            })
            return result

        ccol = need(col_col, 'COL_COL')
        result['col_col'] = ccol
        agg = (df.filter(col(rcol).is_not_null() & col(ccol).is_not_null())
                 .group_by(rcol, ccol).agg(count(lit(1)).alias('N')).to_pandas())
        agg[rcol] = agg[rcol].astype(str)
        agg[ccol] = agg[ccol].astype(str)
        table = agg.pivot_table(index=rcol, columns=ccol, values='N', aggfunc='sum', fill_value=0)
        table = table.sort_index().sort_index(axis=1)
        if table.shape[0] < 2 or table.shape[1] < 2:
            raise ValueError('each variable needs at least 2 categories')

        obs = table.to_numpy(dtype=float)
        n = obs.sum()
        is_2x2 = obs.shape == (2, 2)
        chi2, p, dof, exp = stats.chi2_contingency(obs, correction=False)
        rows, cols_ = list(table.index), list(table.columns)
        rsum, csum = obs.sum(axis=1), obs.sum(axis=0)

        low_ratio = float((exp < 5).sum()) / exp.size
        k = min(obs.shape) - 1
        cramers_v = math.sqrt(chi2 / (n * k)) if n > 0 and k > 0 else None

        cells = {}
        for i, r in enumerate(rows):
            cells[r] = {}
            for j, c in enumerate(cols_):
                e = exp[i, j]
                denom = math.sqrt(e * (1 - rsum[i] / n) * (1 - csum[j] / n))
                cells[r][c] = {
                    'observed': int(obs[i, j]),
                    'expected': _f(e),
                    'row_pct': _f(obs[i, j] / rsum[i] * 100) if rsum[i] else None,
                    'adj_residual': _f((obs[i, j] - e) / denom) if denom > 0 else None,
                }

        result.update({
            'n': int(n),
            'rows': rows, 'columns': cols_,
            'cells': cells,
            'pearson': {'chi2': _f(chi2), 'df': int(dof), 'p_value': _f(p)},
            'cramers_v': _f(cramers_v),
            'association_strength': _strength(cramers_v),
            'expected_lt5_ratio': _f(low_ratio),
        })

        p_rec, rec = p, 'pearson'
        if is_2x2:
            chi2_y, p_y, _, _ = stats.chi2_contingency(obs, correction=True)
            odds, p_f = stats.fisher_exact(obs.astype(int))
            result['yates'] = {'chi2': _f(chi2_y), 'df': 1, 'p_value': _f(p_y)}
            result['fisher_exact'] = {'odds_ratio': _f(odds), 'p_value': _f(p_f)}
            if low_ratio > 0:
                p_rec, rec = p_f, 'fisher_exact'
        elif low_ratio > 0.2:
            result['warning'] = '기대빈도 5 미만 셀이 20%를 넘어 카이제곱 근사가 부정확할 수 있음 (범주 병합 고려)'

        result['recommended_test'] = rec
        result['decision'] = _decision(_f(p_rec), alpha)
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

### 5.1 독립성 검정: 그룹 × 구매여부

```sql
CALL KWS_DB.STATISTICS.RUN_CHISQ(
  TABLE_NAME => 'KWS_DB.STATISTICS.SAMPLE_CHISQ',
  ROW_COL => 'AB_GROUP', COL_COL => 'PURCHASED');
```

| 항목 | 값 |
|---|---|
| 구매율 (A / B) | 16.8% / 34.5% |
| Pearson χ² / df / p | 24.69 / 1 / 6.75 × 10⁻⁷ |
| Yates 보정 p | 1.09 × 10⁻⁶ |
| Fisher 정확검정 p | 9.23 × 10⁻⁷ |
| 오즈비 | 2.61 |
| Cramér's V | 0.20 (약함) |
| 조정잔차 (B, Y) | +4.97 |
| 기대빈도 5 미만 셀 | 없음 → Pearson 권장 |
| 판정 | **유의함** |

**해석:** 프로모션 그룹과 구매 여부는 관련이 있습니다. B그룹의 구매율(34.5%)이 A그룹(16.8%)의 약 2배이고, 오즈비로 보면 B그룹의 구매 오즈가 A그룹의 약 2.6배입니다. 조정잔차가 +4.97이므로 "B그룹의 구매(Y)"가 기대보다 크게 많은 셀입니다. 연관 강도 자체는 '약함'인데, 구매율 차이가 의미 있더라도 전체 변동 중 그룹이 설명하는 부분은 크지 않다는 뜻입니다.

> 2×2 표이므로 Pearson, Yates, Fisher 결과가 모두 반환됩니다. 기대빈도가 모두 5 이상이어서 Pearson 결과로 판정했고, 세 검정의 결론도 모두 같습니다.

### 5.2 독립성 검정: 그룹 × 채널

```sql
CALL KWS_DB.STATISTICS.RUN_CHISQ(
  TABLE_NAME => 'KWS_DB.STATISTICS.SAMPLE_CHISQ',
  ROW_COL => 'AB_GROUP', COL_COL => 'CHANNEL');
```

| 항목 | 값 |
|---|---|
| χ² / df / p | 0.73 / 2 / 0.694 |
| Cramér's V | 0.03 (거의 없음) |
| 조정잔차 최댓값 | 0.84 (모두 ±2 이내) |
| 판정 | **유의하지 않음** |

**해석:** 그룹에 따라 이용 채널 분포가 다르다고 볼 근거가 없습니다. 무작위 배정이 잘 되었다는 근거로도 쓸 수 있습니다. 실무에서는 이런 **사전 균형 점검(balance check)** 에 자주 쓰입니다.

### 5.3 적합도 검정: 방문요일이 균등한가

```sql
CALL KWS_DB.STATISTICS.RUN_CHISQ(
  TABLE_NAME => 'KWS_DB.STATISTICS.SAMPLE_CHISQ',
  ROW_COL => 'VISIT_DAY', TEST_TYPE => 'goodness_of_fit');
```

| 요일 | 관측 | 기대 | 표준화잔차 |
|---|---|---|---|
| MON | 65 | 85.7 | −2.24 |
| TUE | 72 | 85.7 | −1.48 |
| WED | 62 | 85.7 | −2.56 |
| THU | 70 | 85.7 | −1.70 |
| FRI | 88 | 85.7 | +0.25 |
| SAT | 123 | 85.7 | **+4.03** |
| SUN | 120 | 85.7 | **+3.70** |

| 항목 | 값 |
|---|---|
| χ² / df / p | 46.64 / 6 / 2.21 × 10⁻⁸ |
| 판정 | **유의함** (균등하지 않음) |

**해석:** 방문이 요일별로 균등하지 않습니다. 토·일요일이 기대보다 크게 많고, 월·수요일이 적습니다.

### 5.4 적합도 검정: 방문요일이 설계 비율을 따르는가

```sql
CALL KWS_DB.STATISTICS.RUN_CHISQ(
  TABLE_NAME => 'KWS_DB.STATISTICS.SAMPLE_CHISQ',
  ROW_COL => 'VISIT_DAY', TEST_TYPE => 'goodness_of_fit',
  EXPECTED_PROBS => '{"MON":0.12,"TUE":0.12,"WED":0.12,"THU":0.12,"FRI":0.14,"SAT":0.19,"SUN":0.19}');
```

| 항목 | 값 |
|---|---|
| χ² / df / p | 3.34 / 6 / 0.765 |
| 표준화잔차 범위 | −1.18 ~ +0.84 (모두 ±2 이내) |
| 판정 | **유의하지 않음** (설계 비율과 일치) |

**해석:** 관측된 요일 분포가 기대 비율과 다르다고 볼 근거가 없습니다. 같은 데이터라도 **어떤 기대분포와 비교하느냐에 따라 결론이 달라진다**는 점을 보여주는 예입니다.

### 5.5 설계값과 비교

| 검정 | 설계상 정답 | 측정 결과 | 일치 |
|---|---|---|---|
| 그룹 × 구매여부 | 관련 있음 (18% vs 30%) | 16.8% vs 34.5%, p = 6.75 × 10⁻⁷ | ✔ |
| 그룹 × 채널 | 관련 없음 | p = 0.694 | ✔ |
| 방문요일 vs 균등 | 균등하지 않음 | p = 2.21 × 10⁻⁸ | ✔ |
| 방문요일 vs 설계 비율 | 일치 | p = 0.765 | ✔ |

네 가지 검정 모두 의도한 결론과 일치합니다.

## 6. 에이전트에 연결할 때 팁

Cortex Agent에 custom tool로 등록할 때는 **도구 설명(description)** 이 호출 정확도를 좌우합니다. 예시는 다음과 같습니다.

```text
범주형 변수에 대한 카이제곱 검정을 수행한다.
- 두 범주형 변수가 서로 관련 있는지, 또는 집단별 범주 분포가 같은지:
  TEST_TYPE='independence', ROW_COL과 COL_COL에 두 컬럼
- 한 범주형 변수의 분포가 균등하거나 특정 비율을 따르는지:
  TEST_TYPE='goodness_of_fit', ROW_COL에 컬럼,
  특정 비율이면 EXPECTED_PROBS에 {"범주":비율} JSON (생략하면 균등분포)
- 특정 기간이나 조건만 분석할 때는 FILTER_COL/FILTER_VALUE 사용
결과의 decision과 p_value로 유의성을 판단하고, cramers_v로 연관 강도를,
adj_residual 또는 std_residual이 ±2를 넘는 셀로 어느 범주가 차이를 만드는지 설명한다.
recommended_test가 fisher_exact이면 fisher_exact 결과를 기준으로 답한다.
수치형 변수의 평균 비교에는 이 도구가 아니라 RUN_TTEST를 사용한다.
```

함께 챙겨야 할 것들입니다.

- **t검정 도구와 구분:** 두 도구를 함께 붙이면 "A/B 그룹 비교"라는 질문에서 어떤 도구를 써야 할지 헷갈릴 수 있습니다. **평균 비교(구매금액)는 RUN_TTEST, 비율·분포 비교(구매 여부, 채널)는 RUN_CHISQ**라는 기준을 instructions에 명시하세요.
- **범주 값 사전:** 에이전트가 `Y/N`, `MON~SUN` 같은 실제 값을 알아야 `EXPECTED_PROBS`를 정확히 만들 수 있습니다. instructions나 시맨틱 뷰의 sample values로 알려주세요.
- **권한:** 에이전트를 사용하는 역할에 프로시저 USAGE 권한과 대상 테이블 SELECT 권한이 필요합니다.
- **범주가 너무 많은 컬럼:** 고객 ID처럼 범주가 수천 개인 컬럼으로 검정하면 의미 없는 결과가 나오고, 기대빈도 경고도 발생합니다. 범주형 컬럼만 대상으로 하도록 안내하세요.
- **인과 해석 주의:** 카이제곱 검정은 **관련성**을 보여줄 뿐 인과관계를 증명하지는 않습니다. 이 샘플은 무작위 배정 실험이라 인과로 해석할 수 있지만, 관찰 데이터라면 해석에 주의하도록 응답 지침에 넣는 것이 좋습니다.

## 7. 마무리

- 교차표는 SQL로 되지만, **p-value, Fisher 정확검정, 연관 강도, 잔차 분석**은 Python 저장 프로시저로 해결했습니다.
- **집계는 Snowflake에서, 검정은 Python에서** 나눠서 처리해 대용량 데이터에도 쓸 수 있게 만들었습니다.
- t검정 도구(`RUN_TTEST`)와 함께 쓰면 **수치형과 범주형 비교를 모두** 자연어로 처리할 수 있습니다.
- 다음 단계로 **ANOVA(세 집단 이상 평균 비교)**, **상관 검정**, **정규성 검정** 도구도 같은 방식으로 만들 수 있습니다.

---

**환경:** Snowflake, Python 3.11 런타임, `scipy` / `numpy` / `pandas` (Snowflake Anaconda 채널)
