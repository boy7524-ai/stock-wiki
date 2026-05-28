# Stock Wiki — 데이터 fetch 신뢰성 개선 설계

작성일: 2026-05-28
대상 리포: `boy7524-ai/stock-wiki`
선행 spec: `2026-05-28-stock-wiki-trading-journal-dashboard-design.md`

## 1. 목적 & 배경

2026-05-28 21시 routine 실행 결과(`a214295`)에서 다음 두 가지 실패가 동시 발생:

- 📡 **Naver Finance 데이터 갱신 실패** — 11개 한국 종목 전체, `finance.naver.com` WebFetch 접근 불가
- 📡 **Yahoo Finance Japan 데이터 갱신 실패** — 고베물산(3038.T), `finance.yahoo.co.jp` 동일 차단

결과적으로 Dashboard `§2 종목 펀더멘털` 표의 6개 종목(코웨이 4/3, 동성화인텍 4/X, 한화에어로스페이스 5/11, 케이엠더블유 5/12, 실리콘투 5/8, 고베물산 3/13)이 **수 주~수 개월 전 값**을 표시했고, WebSearch fallback이 캐시된 과거 검색결과를 가져온 것이 원인이다.

본 spec은 cloud routine의 가격·밸류에이션 fetch 단계를 **다중 소스 cascade**로 재설계해 단일 소스 실패가 전체 데이터 노후화로 이어지지 않도록 한다.

## 2. 범위

### 포함
- 가격(`현재가`, `전일대비`), Trailing PER, 시가총액, 환율(USD/KRW, JPY/KRW)의 cascade fetch
- Forward PER, 목표가 컨센서스의 fallback 소스 추가
- 실패 시 frontmatter 갱신 정책 및 Dashboard 경고 가시화
- routine prompt의 가격 fetch 블록 교체
- Dashboard `§2` 표의 ⚠️ "오늘 데이터 아님" 마크를 routine이 자동 부착

### 제외 (별도 spec 또는 후속)
- 매매일지 입력 UX 개선 — 별도 spec으로 후속 작업
- 매매일지 스키마, 거래 컬럼, Dashboard 표 구조 변경
- 영업이익증가율 fetch 자동화 — 분기 실적 발표 트리거(별도)
- 종목 추가/제거 워크플로우
- 실시간(장중) 가격, 차트 시각화
- 로컬 스크립트 전환

## 3. 데이터 소스 cascade

### Primary: Yahoo Finance 글로벌
- URL: `https://finance.yahoo.com/quote/<TICKER>`
- 종목코드 매핑:
  - KOSPI: `<6자리>.KS` (예: SK하이닉스 `000660.KS`)
  - KOSDAQ: `<6자리>.KQ` (예: 케이엠더블유 `032500.KQ`)
  - 일본: `<코드>.T` (예: 고베물산 `3038.T`)
- 추출 필드: `현재가`, `전일대비(%)`, `시가총액`, `Trailing PER`, native 통화
- 환율: `USDKRW=X`, `JPYKRW=X` 동일 도메인

### Fallback 1: Investing.com 한국어
- URL: `https://kr.investing.com/equities/<slug>` (예: `samsung-electronics-co-ltd`)
- 종목별 slug는 routine prompt 내 매핑 표로 관리 (12개 고정)
- 추출 필드: `현재가`, `Forward PER`, `목표가 컨센서스` (Primary가 못 채운 필드 보강)

### Fallback 2: WebSearch 정밀 쿼리
- 쿼리 형식: `"<종목명> 종가 <오늘 날짜 YYYY-MM-DD> 한국거래소"` (일본 종목은 `"<종목명> 終値 YYYY-MM-DD"`)
- 결과의 상위 3개 페이지를 검토해 오늘 날짜의 종가가 명시된 페이지에서만 추출
- 날짜 일치하지 않으면 실패 처리 (전일 값 유지)
- 추출 필드: `현재가`만 (최후수단)

### cascade 동작 규칙
1. Primary 시도 → 성공하면 거기서 종료, `data_source=yahoo` 기록
2. Primary 실패(HTTP 4xx/5xx, 빈 응답, 추출 실패) → Fallback 1 시도, 성공 시 `data_source=investing`
3. Fallback 1도 실패 → Fallback 2 시도, 성공 시 `data_source=websearch`
4. 모두 실패 → frontmatter 가격 필드 갱신 안 함, `data_source=failed`

여기서 "실패"는 다음을 모두 포함:
- HTTP 4xx/5xx 응답 또는 timeout
- HTTP 200이지만 가격 추출 패턴 매치 실패
- 추출된 값이 직전 거래일과 비현실적 차이(±50% 이상) — sanity check

## 4. 필드 매핑 & 갱신 정책

| 필드 | Primary | Fallback 1 | Fallback 2 | 모두 실패 |
|------|---------|-----------|-----------|----------|
| `현재가` | Yahoo | Investing | WebSearch | 갱신 안 함 |
| `전일대비` | Yahoo | Investing | (계산: 오늘/어제) | 갱신 안 함 |
| `시가총액` | Yahoo | Investing | — | 갱신 안 함 |
| `PER` (Trailing) | Yahoo | Investing | — | 갱신 안 함 |
| `PER_forward` | Investing (우선) | Yahoo | — | 이전 값 유지 |
| `목표가_컨센서스` | Investing | (없음) | — | 이전 값 유지 |
| `목표가_괴리율` | 계산: `(목표가 - 현재가) / 현재가 × 100` | — | — | 현재가 갱신 안 됐으면 갱신 안 함 |
| `업데이트` | 오늘 날짜 (가격 갱신 성공 시) | 동일 | 동일 | 어제 날짜 유지 |
| `영업이익증가율` | (이 spec 범위 외, 분기 실적 트리거) | — | — | — |

### Forward PER의 Primary
`PER_forward`만 Investing.com을 Primary로 둔다. Yahoo Finance는 한국 종목에서 Forward PER을 잘 노출하지 않기 때문. Investing.com에서 못 가져오면 Yahoo의 `forwardPE` 필드를 시도, 둘 다 실패면 이전 값 유지.

### 환율
- `USD/KRW`, `JPY/KRW` 매일 한 번씩 Yahoo Finance에서 fetch
- 환율 fetch 실패 시 전일 환율 사용 + Dashboard 경고에 `💱 환율 갱신 실패 (전일 환율 사용)`
- 외화 종목(현재 고베물산)의 `현재가`는 native 통화(JPY)로 fetch → 환율로 KRW 환산해 저장

## 5. 실패 정책 & 경고 가시화

### frontmatter 동작

```yaml
# 성공 케이스 (Primary)
현재가: 245000
전일대비: 1.2
업데이트: 2026-05-29
data_source: yahoo  # 신규 필드

# Fallback 1 성공
현재가: 245000
전일대비: 1.2
업데이트: 2026-05-29
data_source: investing

# 모두 실패 (전 거래일 값 유지)
현재가: 245000          # 변경 없음
전일대비: 1.2           # 변경 없음
업데이트: 2026-05-28    # 변경 없음 (전 거래일)
data_source: failed     # 표시만 변경
```

신규 필드 `data_source`를 frontmatter에 추가. 값은 `yahoo` / `investing` / `websearch` / `failed` / `holiday` 중 하나.

### Dashboard 경고 섹션

기존 경고 패턴 유지 + cascade-specific 추가:

| 케이스 | 표시 |
|--------|------|
| Primary 성공 | (경고 없음) |
| Fallback 1/2 성공 | `ℹ️ [종목] fallback 소스 사용 (<source>) — Primary 실패 원인 확인 필요` |
| 모두 실패 | `📡 [종목] 모든 소스 실패 — 전일(YYYY-MM-DD) 값 표시 중` |
| 휴장 | `📅 [시장] 휴장 — 종가 미갱신` |
| 환율 실패 | `💱 USD/KRW 또는 JPY/KRW 환율 갱신 실패 — 전일 환율 사용` |
| 비현실적 변동 차단 | `⚠️ [종목] 추출값 sanity check 실패 (±50% 초과) — 갱신 안 함` |

### Dashboard §2 표의 ⚠️ 마크 자동화

현재 Dashboard는 사람이 판단해 "최근 거래일 데이터" 종목 옆에 ⚠️를 붙였다. 변경 후:
- routine이 frontmatter의 `업데이트` 필드 ≠ 오늘 날짜인 경우 자동으로 ⚠️ 부착
- 휴장일에는 ⚠️ 부착 안 함 (정상)
- ⚠️ 옆에 작은 글씨로 `(업데이트: YYYY-MM-DD)` 추가

## 6. routine prompt 변경 사항

### 변경 범위
- 기존 prompt의 Step A(가격 fetch) 블록을 cascade 로직으로 교체
- Step B(frontmatter merge): `data_source` 필드 신규 기록만 추가
- Step C(매매일지 파싱), E(commit) **변경 없음**
- Step D(Dashboard 재생성): §5의 ⚠️ 자동 부착 로직과 경고 섹션 패턴 추가만
- 종목코드/섹터/시장/통화 매핑 표는 routine prompt에 기존대로 유지

### 새 Step A 블록

routine prompt에 다음 형식으로 들어갈 (의사코드, 실제 prompt는 plan 단계에서 작성):

```
Step A: 12종목 가격·밸류에이션 fetch (cascade)

각 종목에 대해 아래 순서로 시도:

A.1 Primary — Yahoo Finance 글로벌
   WebFetch: https://finance.yahoo.com/quote/<ticker>
   ticker 매핑: KOSPI→.KS, KOSDAQ→.KQ, 일본→.T
   추출: regularMarketPrice, regularMarketChangePercent, marketCap, trailingPE, currency
   sanity: 추출값 vs 이전 frontmatter 값 ±50% 이내
   성공 시: data_source=yahoo, Step B로

A.2 Fallback 1 — Investing.com 한국어
   slug 매핑 표 사용 (12개 사전 정의)
   WebFetch: https://kr.investing.com/equities/<slug>
   추출: 현재가, Forward PER, 목표가 컨센서스
   성공 시: data_source=investing, Step B로

A.3 Fallback 2 — WebSearch
   쿼리: "<종목명> 종가 <오늘> 한국거래소" (한국) / "<종목명> 終値 <오늘>" (일본)
   상위 3개 결과에서 오늘 날짜 종가 명시한 것만 채택
   성공 시: data_source=websearch, Step B로

A.4 모두 실패
   data_source=failed
   frontmatter 가격 필드는 갱신 안 함
   Dashboard 경고 섹션에 추가

환율 fetch (cascade와 독립):
   Yahoo Finance USDKRW=X, JPYKRW=X
   실패 시 frontmatter 갱신 안 함 + 경고
```

### Investing.com slug 매핑 표 (routine prompt 내 고정)

12종목 각각의 Investing.com URL slug:

| 종목 | slug |
|------|------|
| SK하이닉스 | `sk-hynix-inc` |
| HD현대중공업 | `hyundai-heavy-industries` |
| 삼성바이오로직스 | `samsung-biologics` |
| 한화에어로스페이스 | `hanwha-aerospace` |
| 메리츠금융지주 | `meritz-financial-group` |
| 대한항공 | `korean-air-lines` |
| 코웨이 | `coway` |
| 풍산 | `poongsan` |
| 동성화인텍 | `dongsung-finetec` |
| 케이엠더블유 | `kmw-co-ltd` |
| 실리콘투 | `silicon2` |
| 고베물산 | `kobe-bussan-co-ltd` |

위 slug는 plan Task 1에서 routine 첫 수동 실행 시 검증한다. slug가 틀린 종목은 plan에서 수정 후 매핑 표 갱신.

## 7. 검증 방식

### 단계별 검증

| 단계 | 방법 | 기준 |
|------|------|------|
| 1. routine prompt 업데이트 후 | `claude routines list`로 prompt에 cascade 키워드 포함 확인 | "Step A.1", "A.2", "A.3", "fallback", "Investing" 모두 포함 |
| 2. routine 1회 수동 실행 | `claude -p "routine ID ... 지금 실행"` 후 약 5~7분 대기 | commit/push 성공 |
| 3. Dashboard 검증 | `data_source` 분포 확인 | yahoo ≥ 8개 / investing ≤ 3개 / websearch ≤ 1개 / failed ≤ 1개 |
| 4. 종목 frontmatter 검증 | 12종목 `업데이트` 필드 = 오늘 날짜 | 휴장 종목 제외 12개 모두 |
| 5. Dashboard 경고 가시화 | failed 종목이 있다면 경고 섹션에 명시 | 표시되지 않으면 routine 수정 |

### Primary 성공률 임계
- 첫 routine 실행에서 Primary 성공률 < 8개면 cascade 순서 재조정 검토
- 예: Yahoo가 거의 실패하면 Investing.com을 Primary로 승격

### 1주일 누적 모니터링
- 7일간 매일 21시 routine 결과의 `data_source` 분포 추적
- Primary 성공률 < 70% 지속 시 Primary 교체

## 8. 위험 & 완화

| 위험 | 완화 |
|------|------|
| Yahoo Finance도 routine 환경에서 차단됨 | Fallback 1/2가 동작하므로 데이터 노후화는 막힘. 1주 후 Primary 재선정 |
| Investing.com slug 변경 또는 종목명 미등록 | plan 단계에서 12개 slug 사전 검증, 1개 틀려도 cascade가 다음으로 넘어감 |
| WebSearch 결과 캐시 노후화 | 쿼리에 오늘 날짜 명시, 결과 페이지에서 날짜 일치 검증 |
| Yahoo의 한국 종목 sparse 데이터 (PER N/A 등) | 필드별 cascade — Forward PER은 Investing이 Primary |
| 비현실적 추출값 (HTML 구조 변경 후 잘못된 셀 추출) | ±50% sanity check + 실패 처리 |
| 환율 fetch 실패 → 외화 종목 환산 오류 | 전일 환율 fallback + Dashboard 경고 가시화 |
| 휴장일에 "fetch 실패"로 잘못 인식 | 시장별 휴장 캘린더 — 한국 KRX, 일본 JPX 공식 휴장일 기준 (routine이 인식) |
| `data_source` 필드 추가로 frontmatter 호환성 | 기존 reader(Obsidian, Dataview)는 모르는 필드 무시 — 영향 없음 |

## 9. 후속 spec (매매일지 UX)

본 spec과 별도로 다음 작업을 spec → plan → 구현 사이클로 진행:

- 매매일지 거래 입력 UX 개선
- 후보 방향: 빠른 입력 라인(routine 파싱) / 별도 inbox 파일 / Obsidian QuickAdd 플러그인 / 로컬 CLI
- 이번 spec 완료 후 brainstorming 재개

## 10. 비목표 (Out of Scope)

- 로컬 스크립트 전환 (cloud routine 유지 결정)
- 매매일지 입력 UX (별도 spec)
- 실시간 가격, 차트 시각화
- 분기 실적 자동 감지
- 종목 추가/제거 워크플로우
- KIS OpenAPI 등 인증 필요 API 도입
