# 런던 자치구 CO2 배출량 EDA 정리

## 0. 데이터 출처
https://public.tableau.com/app/learn/sample-data

---

## 1. 데이터 구조

### 원본 파일 (`carbon-emissions-borough.xls`)
시트 3개로 구성:

| 시트 | 내용 |
|---|---|
| `Metadata` | 데이터 출처, 설명 등 메타정보 |
| `TOTAL` | 자치구 경계 내 **모든** CO2 배출량 (LULUCF 포함) |
| `SUBSET` | 지방자치단체가 **직접 영향을 줄 수 있는 범위**의 배출량 (LULUCF 제외) |

원본은 다음과 같은 **와이드 포맷**: 행 = 지역(자치구/지역/국가), 열 = 부문(Sector) × 연도(Year)가 다단 헤더로 결합

부문 그룹: `Industry and Commercial`, `Domestic`, `Transport`, `Grand total`, `Population ('000s)`, `Per Capita Emissions (t)` (TOTAL 시트는 여기에 `LULUCF Net Emissions`가 추가로 있음)
연도 범위: **2005 ~ 2014년** (10개년)

원본 SUBSET 시트 앞부분 예시 (일부 컬럼만 표시):

| Code | Name | Industry and Commercial\|2005 | Domestic\|2005 | Transport\|2005 | Grand total\|2005 | Population\|2005 |
|---|---|---|---|---|---|---|
| E09000001 | City of London | 1545.66 | 20.35 | 65.32 | 1631.33 | 7.131 |
| E09000002 | Barking and Dagenham | 268.35 | ... | ... | ... | ... |

전체 행 구성은 **런던 33개 자치구 + 잉글랜드 9개 지역(North East 등) + 런던 전체 + England/Scotland/Wales/Northern Ireland/UK 집계행**이 섞여 있어서, 분석 시 `Code`가 `E09`로 시작하는 33개 자치구만 필터링해서 사용함.

### 가공한 Long format (`co2_emissions_long.csv`)

와이드 포맷은 시각화·집계가 어려워서, `[Code, Name, Year, Sector, Emissions_kt, Population_000s, PerCapita_Emissions_t]` 구조의 long format으로 변환함.

- 33개 자치구 × 10개 연도 × 4개 부문(Industry/Domestic/Transport/Grand total) = **1,320행**
- Population, Per Capita Emissions은 부문과 무관하게 Code+Year 기준으로 값이 동일하게 반복 삽입 (조인 결과)

앞 8줄 예시:

| Code | Name | Year | Sector | Emissions_kt | Population_000s | PerCapita_Emissions_t |
|---|---|---|---|---|---|---|
| E09000001 | City of London | 2005 | Domestic | 20.35 | 7.131 | 228.77 |
| E09000001 | City of London | 2005 | Grand total | 1631.33 | 7.131 | 228.77 |
| E09000001 | City of London | 2005 | Industry and Commercial | 1545.66 | 7.131 | 228.77 |
| E09000001 | City of London | 2005 | Transport | 65.32 | 7.131 | 228.77 |
| E09000001 | City of London | 2006 | Domestic | 20.40 | 7.254 | 243.34 |
| E09000001 | City of London | 2006 | Grand total | 1765.18 | 7.254 | 243.34 |
| E09000001 | City of London | 2006 | Industry and Commercial | 1679.66 | 7.254 | 243.34 |
| E09000001 | City of London | 2006 | Transport | 65.12 | 7.254 | 243.34 |

---

## 2. TOTAL vs SUBSET — 어떤 것을 쓸 것인가

| | TOTAL | SUBSET |
|---|---|---|
| 포함 범위 | 자치구 경계 내 **모든** 배출 (LULUCF 포함) | 자치구가 **직접 통제 가능한** 배출만 (LULUCF 제외) |
| 용도 | 지역 내 실제 총배출량 파악 | 국가지표(National Indicator 186) 보고용, 정책 효과 비교용 |
| 특징 | 자치구가 어찌할 수 없는 요인(국가 발전소 위치 등)까지 포함돼 지역 간 비교가 왜곡될 수 있음 | 자치구 정책 노력만 반영되므로 지역 간 "감축 성과" 비교에 더 적합 |

**➡ 이 프로젝트는 SUBSET을 사용한다.**
분석 목적이 "자치구별로 어디에 정책을 우선 투입해야 하는가"를 보는 것이므로, 자치구가 통제할 수 없는 배출까지 포함된 TOTAL보다는 **자치구의 실질적 영향권 안의 배출량(SUBSET)** 이 지역 간 공정한 비교와 정책 우선순위 도출에 더 적합하기 때문이다.

---

## 3. 기초 EDA 결과

### 3-1. 데이터 품질
- 결측치: **0건** (long format 기준 7개 컬럼 전부 결측 없음)
- 타입: Year(정수), Emissions_kt/Population_000s/PerCapita_Emissions_t(실수) — 모두 정상 인식됨
- 자치구 수: 33개, 연도 범위: 2005~2014년 (10개년)

### 3-2. 이상치 후보 — City of London
- **총배출량(Grand total) 기준**으로는 City of London 평균(1,522kt)이 나머지 32개 자치구 평균(1,290kt)의 약 **1.2배**로, 절대량 자체는 최고 지역(Westminster 3,184kt)보다도 낮음 → 총량 기준으로는 튀는 이상치가 아님
- 하지만 **인구 1인당 배출량(Per Capita)** 기준으로는 완전히 다른 그림: 2014년 기준 City of London은 **128.3t/인**으로, 2위인 Westminster(10.7t/인)의 **12배**에 달함 (인구가 7천여 명으로 극히 적은데 상업지구 배출은 크기 때문)
- ➡ **결론: City of London은 "총량" 지표에서는 문제없지만 "1인당" 지표를 다룰 땐 반드시 별도 표시하거나 제외 처리 필요**

### 3-3. 부문(Sector)별 기술통계 (자치구×연도 단위, kt)

| Sector | mean | std | min | max |
|---|---|---|---|---|
| Industry and Commercial | 583.4 | 459.3 | 182.7 | 2712.2 |
| Domestic | 474.9 | 148.9 | 16.5 | 870.7 |
| Transport | 238.6 | 83.5 | 50.5 | 544.3 |
| Grand total | 1297.0 | 484.4 | 604.5 | 3564.9 |

→ Industry and Commercial 부문의 표준편차가 가장 커서 자치구 간 산업 구조 차이가 배출량 편차의 주요 원인으로 보임

### 3-4. 런던 전체 배출량 추이 (Grand total 합계, kt)

| 연도 | 배출량 |
|---|---|
| 2005 | 46,159 |
| 2008 | 45,952 |
| 2011 | 39,298 |
| 2014 | 35,103 |

→ 2005년 대비 2014년 **약 24% 감소**. 다만 연도별로 등락이 있어 (2006년 소폭 상승, 2010년 반등 등) 매끄러운 감소가 아니라 변동을 동반한 감소 추세

