# Stock Wiki — 데이터 fetch 신뢰성 개선 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Cloud routine `trig_01SZu6hi1mTygQ7G6ztER4Ch`의 가격·밸류에이션 fetch를 Yahoo Finance 글로벌 → Investing.com 한국어 → WebSearch 정밀쿼리 3단계 cascade로 재설계해, 2026-05-28에 발생한 Naver Finance / Yahoo Finance Japan 차단으로 인한 12종목 중 6개 노후 데이터 표시 문제를 해결한다.

**Architecture:** 기존 cloud routine prompt에서 Step A(가격 fetch) 블록을 cascade 로직으로 교체. Step B(frontmatter merge)에 `data_source` 신규 필드 기록을 추가하고, Step D(Dashboard 재생성)에 ⚠️ 자동 부착·경고 섹션 패턴을 추가한다. 12종목의 Investing.com slug 매핑은 plan Task 2에서 검증해 routine prompt에 임베드.

**Tech Stack:** Claude Code cloud routine (sonnet-4-6), WebFetch (finance.yahoo.com / kr.investing.com), WebSearch, Markdown + YAML frontmatter, `claude` CLI.

**Spec 참조:** `docs/superpowers/specs/2026-05-28-data-fetch-reliability-design.md`

---

## File Structure

| 파일 / 자원 | 책임 | 누가 편집 |
|------------|------|----------|
| Cloud routine `trig_01SZu6hi1mTygQ7G6ztER4Ch` prompt | Step A 교체, Step B/D 부분 수정 | Task 4에서 한 번 업데이트 |
| `20_Stocks/*.md` (12개) frontmatter | `data_source` 키 자동 추가 | Routine 첫 실행 (Task 5) |
| `10_Summary/Dashboard.md` | ⚠️ 자동 부착, 경고 섹션 패턴 변경 | Routine 매일 재생성 |
| `docs/_workspace/2026-05-28-routine-prompt-backup.txt` | 기존 routine prompt 원본 백업 (plan 진행 임시) | Task 1 |
| `docs/_workspace/2026-05-28-investing-slug-mapping.md` | 12종목 Investing slug 검증 결과 | Task 2 |
| `docs/_workspace/2026-05-28-routine-prompt-new.txt` | 새 routine prompt 머지 결과 | Task 3 |

`docs/_workspace/`는 plan 진행 중에만 사용하는 작업 폴더. plan 완료 후 정리(Task 5 Step 7).

---

## Task 1: 현재 routine prompt 백업

**Files:**
- Create: `docs/_workspace/2026-05-28-routine-prompt-backup.txt`

목적: routine prompt를 잘못 수정해 망가뜨릴 위험이 있으므로, 변경 전 전체 prompt를 백업한다.

- [ ] **Step 1: `docs/_workspace/` 디렉토리 생성**

```bash
mkdir -p docs/_workspace
```

- [ ] **Step 2: 현재 routine prompt 조회 + 파일 저장**

다음 명령을 실행. `claude -p`는 새 Claude 세션에 단발성 메시지를 보내 응답만 받는다.

```bash
claude -p "routine ID trig_01SZu6hi1mTygQ7G6ztER4Ch의 현재 prompt 전문을 출력해줘. 절대 요약하지 말고 원문 그대로. prompt 외 부가 설명도 붙이지 마." > docs/_workspace/2026-05-28-routine-prompt-backup.txt
```

- [ ] **Step 3: 백업 내용 검증**

```bash
wc -l docs/_workspace/2026-05-28-routine-prompt-backup.txt
grep -c "12종목\|뉴스\|브리핑\|commit" docs/_workspace/2026-05-28-routine-prompt-backup.txt
```

Expected:
- 줄 수 ≥ 30 (요약본이 아닌 원본임을 확인)
- 키워드 매칭 수 ≥ 3 (routine 핵심 기능이 prompt에 들어있음)

키워드가 없거나 줄 수가 너무 적으면 Step 2의 `claude -p` 결과가 요약된 것. 다시 시도하고 "정말 원문 그대로 줘"라고 강조.

- [ ] **Step 4: Step A(가격 fetch) 블록 위치 식별**

백업 파일을 사람이 직접 읽고 Step A 블록 시작 줄과 끝 줄을 메모. 보통 다음 패턴으로 식별:

- 시작: "Step A:" 또는 "12종목 가격" 또는 "Naver Finance"
- 끝: "Step B:" 또는 "frontmatter merge" 직전

이 줄 번호를 Task 3 Step 1에서 사용.

이 task는 commit 없음 (작업 임시 파일).

---

## Task 2: 12종목 Investing.com slug 검증

**Files:**
- Create: `docs/_workspace/2026-05-28-investing-slug-mapping.md`

목적: spec §6 매핑 표의 12개 slug가 실제 Investing.com에서 존재하는지 검증. 틀린 slug가 있으면 수정.

- [ ] **Step 1: 검증할 12 slug 정리**

다음 표를 기준 매핑으로 시작:

| 종목 | 종목코드 | 가설 slug |
|------|---------|----------|
| SK하이닉스 | 000660 | `sk-hynix-inc` |
| HD현대중공업 | 329180 | `hyundai-heavy-industries` |
| 삼성바이오로직스 | 207940 | `samsung-biologics` |
| 한화에어로스페이스 | 012450 | `hanwha-aerospace` |
| 메리츠금융지주 | 138040 | `meritz-financial-group` |
| 대한항공 | 003490 | `korean-air-lines` |
| 코웨이 | 021240 | `coway` |
| 풍산 | 103140 | `poongsan` |
| 동성화인텍 | 033500 | `dongsung-finetec` |
| 케이엠더블유 | 032500 | `kmw-co-ltd` |
| 실리콘투 | 257720 | `silicon2` |
| 고베물산 | 3038 | `kobe-bussan-co-ltd` |

- [ ] **Step 2: 12개 URL 각각 WebFetch로 접근 가능성 검증**

각 종목에 대해 `https://kr.investing.com/equities/<slug>` 페이지를 WebFetch 도구로 fetch. 각 호출의 결과를 분류:

- 200 OK + 종목명 한국어로 본문에 등장 → ✅ 매핑 OK
- 404 또는 다른 종목으로 리다이렉트 → ❌ slug 수정 필요

WebFetch 실패 케이스에 대해 WebSearch 쿼리: `"<종목명> kr.investing.com equities"`. 검색 결과의 첫 페이지에서 올바른 slug 확인 후 가설 수정.

- [ ] **Step 3: 매핑 결과 파일 저장**

12종목 모두 검증 후 다음 형식으로 `docs/_workspace/2026-05-28-investing-slug-mapping.md` 작성:

```markdown
# Investing.com slug 매핑 검증 (2026-05-28)

| 종목 | 종목코드 | slug | 검증 결과 | 비고 |
|------|---------|------|----------|------|
| SK하이닉스 | 000660 | sk-hynix-inc | ✅ |  |
| ... | ... | ... | ... | ... |

## 미확인/실패 종목
- (있다면 종목명과 원인)
```

- [ ] **Step 4: 실패 slug 처리 결정**

12개 모두 ✅면 Task 3로 진행.

1~2개 ❌면: 해당 종목의 Investing.com Fallback 1 단계 비활성화. cascade에서 Yahoo 실패 시 바로 WebSearch로 점프하도록 routine prompt Task 3에서 표시.

3개 이상 ❌면: brainstorming 단계로 돌아가 Fallback 1 소스 재검토 (예: Daum 금융, KRX 직접 등). 사용자에게 보고.

이 task는 commit 없음 (작업 임시 파일).

---

## Task 3: 새 routine prompt 작성

**Files:**
- Create: `docs/_workspace/2026-05-28-routine-prompt-new.txt`
- Read: `docs/_workspace/2026-05-28-routine-prompt-backup.txt`
- Read: `docs/_workspace/2026-05-28-investing-slug-mapping.md`

목적: 백업한 기존 prompt에서 Step A 블록을 cascade 로직으로 교체하고, Step B/D를 부분 수정한 전체 prompt를 작성.

- [ ] **Step 1: 백업 prompt 복사 → 작업 파일 생성**

```bash
cp docs/_workspace/2026-05-28-routine-prompt-backup.txt docs/_workspace/2026-05-28-routine-prompt-new.txt
```

- [ ] **Step 2: Step A 블록을 cascade 로직으로 교체**

`docs/_workspace/2026-05-28-routine-prompt-new.txt`을 열어 Task 1 Step 4에서 식별한 Step A 블록 전체를 삭제하고, 그 자리에 다음 블록을 정확히 삽입:

```text
## Step A: 12종목 가격·밸류에이션 fetch (cascade)

각 종목에 대해 아래 순서로 시도. 한 단계 성공 시 그 종목은 거기서 종료, 실패 시 다음 단계.

### A.1 Primary — Yahoo Finance 글로벌
WebFetch: https://finance.yahoo.com/quote/<TICKER>
TICKER 매핑 규칙:
- KOSPI: <6자리 종목코드>.KS (예: SK하이닉스 → 000660.KS)
- KOSDAQ: <6자리 종목코드>.KQ (예: 케이엠더블유 → 032500.KQ)
- 일본: <코드>.T (예: 고베물산 → 3038.T)

추출 필드:
- 현재가 (regularMarketPrice)
- 전일대비 % (regularMarketChangePercent)
- 시가총액 (marketCap, 억원 단위 환산)
- Trailing PER (trailingPE)
- Forward PER (forwardPE — 있는 경우만, 없으면 빈 값)
- native 통화 (currency)

Sanity check: 추출 현재가가 이전 frontmatter의 현재가와 ±50% 이내인지 확인. 초과 시 실패 처리.

성공 시: data_source = "yahoo" 기록, 이 종목 fetch 종료. Step B로.

### A.2 Fallback 1 — Investing.com 한국어
WebFetch: https://kr.investing.com/equities/<SLUG>

종목별 SLUG 매핑 (12개 고정):
[Task 2의 docs/_workspace/2026-05-28-investing-slug-mapping.md 검증 결과를 여기 표로 그대로 임베드]

추출 필드:
- 현재가
- Forward PER (있으면)
- 목표가 컨센서스 (있으면)

성공 시: data_source = "investing" 기록, Step B로. Primary가 못 채운 필드도 이 단계에서 채움.

### A.3 Fallback 2 — WebSearch 정밀쿼리
한국 종목 쿼리: "<종목명> 종가 <오늘 YYYY-MM-DD> 한국거래소"
일본 종목 쿼리: "<종목명> 終値 <오늘 YYYY-MM-DD>"

상위 3개 결과 페이지를 검토. 오늘 날짜의 종가가 명시된 페이지에서만 현재가 추출. 날짜 일치하지 않으면 추출 실패 간주.

성공 시: data_source = "websearch" 기록, Step B로.

### A.4 모두 실패
data_source = "failed" 기록.
이 종목의 frontmatter 가격 필드(현재가, 전일대비, 업데이트)는 **갱신하지 않음** (전 거래일 값 유지).
Step D Dashboard 경고 섹션에 다음 라인 추가:
  📡 [종목] 모든 소스 실패 — 전일(YYYY-MM-DD) 값 표시 중

### A.5 환율 fetch (cascade와 독립)
WebFetch: https://finance.yahoo.com/quote/USDKRW=X
WebFetch: https://finance.yahoo.com/quote/JPYKRW=X
실패 시: 전일 frontmatter 환율 사용 + Dashboard 경고에 "💱 USD/KRW 또는 JPY/KRW 환율 갱신 실패" 추가.

### A.6 휴장 인식
12종목 중 한국 11개의 가격 fetch가 11개 모두 실패하고 fetch 응답에 "휴장" 또는 "장 마감" 키워드가 나타나면 한국 시장 휴장으로 판정.
휴장 시 data_source = "holiday", frontmatter 가격 필드 갱신 안 함 (정상), Dashboard 헤더에 "📅 KOSPI/KOSDAQ 휴장 — 종가 미갱신" 표시.
일본 시장(고베물산)도 동일 로직.
```

- [ ] **Step 3: Step A에 Investing slug 매핑 표 임베드**

Task 2에서 작성한 `docs/_workspace/2026-05-28-investing-slug-mapping.md`의 매핑 표를 새 prompt Step A.2의 "[Task 2의 ... 임베드]" 자리에 그대로 복사. ❌ 종목이 있다면 그 행에는 "Fallback 1 비활성화 — A.3으로 점프" 비고를 표시.

- [ ] **Step 4: Step B (frontmatter merge) 수정**

기존 Step B 블록에서 "각 종목 noteflow의 frontmatter ... 갱신" 부분을 찾아, 마지막 줄에 다음을 추가:

```text
신규 필드 data_source를 frontmatter에 기록한다. 값은 yahoo / investing / websearch / failed / holiday 중 하나. 기존 frontmatter에 data_source 키가 없으면 신규로 추가.
```

- [ ] **Step 5: Step D (Dashboard 재생성) 수정**

기존 Step D 블록의 §2 표 생성 부분에 다음 로직 추가:

```text
§2 표의 현재가 컬럼에서 frontmatter "업데이트" 필드가 오늘 날짜와 다르면 (휴장 제외):
- 현재가 옆에 " ⚠️" 자동 부착
- 작은 글씨로 "(업데이트: YYYY-MM-DD)" 추가

기존 prompt의 "수동 ⚠️ 부착" 안내문이 있으면 삭제 (이제 routine이 자동 처리).
```

기존 Step D의 경고 섹션 생성 부분에 다음 케이스 패턴 추가:

```text
- 종목별로 frontmatter data_source 값 확인:
  - investing → "ℹ️ [종목] fallback 소스 사용 (investing) — Primary 실패 원인 확인 필요"
  - websearch → "ℹ️ [종목] fallback 소스 사용 (websearch) — Primary, Fallback 1 모두 실패"
  - failed → "📡 [종목] 모든 소스 실패 — 전일(<업데이트 날짜>) 값 표시 중"
- holiday는 헤더 표시이므로 종목별 경고 라인 만들지 않음.
- yahoo는 정상이므로 경고 라인 없음.
```

- [ ] **Step 6: 새 prompt 정합성 검증**

```bash
# 핵심 키워드 모두 포함 확인
for kw in "A.1" "A.2" "A.3" "Yahoo" "Investing" "WebSearch" "data_source" "cascade" "fallback"; do
  echo -n "$kw: "
  grep -c "$kw" docs/_workspace/2026-05-28-routine-prompt-new.txt
done

# Step A~E 모두 존재 확인
for step in "Step A" "Step B" "Step C" "Step D" "Step E"; do
  echo -n "$step: "
  grep -c "$step" docs/_workspace/2026-05-28-routine-prompt-new.txt
done

# 기존 prompt에서 무엇이 사라졌는지 (Step B/C/E 핵심 키워드는 유지되어야)
grep -c "매매일지\|Dashboard\|commit" docs/_workspace/2026-05-28-routine-prompt-new.txt
```

Expected:
- 모든 키워드 count ≥ 1
- Step A~E 각 count ≥ 1
- 마지막 grep count ≥ 3 (기존 routine 기능 유지)

검증 실패 시 새 prompt 재작성.

- [ ] **Step 7: backup vs new diff 확인**

```bash
diff docs/_workspace/2026-05-28-routine-prompt-backup.txt docs/_workspace/2026-05-28-routine-prompt-new.txt | wc -l
```

Expected: 줄 수 50~300 사이. 너무 적으면 변경이 안 됐거나, 너무 많으면 의도하지 않은 부분이 바뀜 — 사람이 diff 직접 검토.

이 task는 commit 없음 (작업 임시 파일).

---

## Task 4: routine prompt 업데이트 실행

**Files:**
- Read: `docs/_workspace/2026-05-28-routine-prompt-new.txt`

목적: cloud routine config의 prompt를 새 내용으로 교체.

- [ ] **Step 1: 새 prompt 파일을 변수로 읽기**

```bash
NEW_PROMPT=$(cat docs/_workspace/2026-05-28-routine-prompt-new.txt)
```

`$NEW_PROMPT` 변수에 새 prompt 전체가 담김.

- [ ] **Step 2: routine prompt 업데이트 명령 실행**

```bash
claude -p "routine ID trig_01SZu6hi1mTygQ7G6ztER4Ch의 prompt를 다음 내용으로 정확히 교체해줘. 다른 설정(스케줄, 도구 권한, 모델)은 그대로 유지. 다음 줄부터가 새 prompt 전문이다:

$NEW_PROMPT"
```

응답으로 "업데이트 완료" 또는 routine ID 확인 메시지가 와야 한다.

- [ ] **Step 3: 업데이트 검증 — prompt 키워드 포함 여부**

```bash
claude -p "routine ID trig_01SZu6hi1mTygQ7G6ztER4Ch의 현재 prompt에 다음 키워드들이 모두 들어있는지 yes/no로 답해줘: 'Step A.1', 'Step A.2', 'Step A.3', 'Yahoo Finance 글로벌', 'Investing.com', 'data_source', 'cascade'"
```

Expected: 모든 키워드 yes.

하나라도 no면 Step 2 재실행 (prompt가 truncate되었거나 잘못 머지됨).

- [ ] **Step 4: routine config 메타 검증**

```bash
claude -p "routine ID trig_01SZu6hi1mTygQ7G6ztER4Ch의 다음 정보를 출력: 스케줄, 다음 실행 시간, enabled 여부, 모델, 사용 가능 도구 목록."
```

Expected:
- 스케줄: 매일 21:01 KST (또는 기존과 동일)
- 다음 실행 시간: 오늘 21:01 또는 내일 21:01
- enabled: true
- 모델: sonnet-4-6 (또는 기존과 동일)
- 도구: WebFetch, WebSearch, Bash 등 포함

스케줄/모델이 바뀌었다면 Step 2 명령에 "다른 설정은 그대로 유지"가 제대로 반영 안 됨. 해당 항목만 원래대로 복원 명령 실행.

이 task는 repo commit 없음 (cloud config 변경).

---

## Task 5: routine 1회 수동 실행 + 검증

목적: 새 prompt가 실제로 동작하는지 확인하고, cascade 분포가 spec §7 기준을 만족하는지 평가.

- [ ] **Step 1: routine 즉시 1회 실행 요청**

```bash
claude -p "routine ID trig_01SZu6hi1mTygQ7G6ztER4Ch을 지금 한 번 실행시켜줘. 완료까지 기다리지 말고 실행만 트리거하고 응답해줘."
```

응답: 실행 시작 확인.

예상 소요: 7~10분 (12종목 cascade fetch 추가로 기존보다 길어짐).

- [ ] **Step 2: 약 10분 대기 후 원격 변경사항 pull**

```bash
sleep 600
git fetch origin
git log --oneline origin/main..HEAD HEAD..origin/main
git pull --rebase --autostash
git log --oneline -3
```

Expected: 새 commit이 보임 (예: `daily: 2026-05-2X 브리핑 및 종목 업데이트`).

새 commit이 없으면 routine 실행이 아직 안 끝났거나 실패. 추가로 5분 더 대기 후 재시도. 20분 넘어도 없으면 routine 로그 확인.

- [ ] **Step 3: 12종목 data_source 분포 집계**

```bash
echo "=== data_source 분포 ==="
for f in 20_Stocks/*.md; do
  ds=$(grep "^data_source:" "$f" | sed 's/data_source: *//')
  echo "$(basename "$f" .md): ${ds:-없음}"
done | sort

echo ""
echo "=== 분포 카운트 ==="
for f in 20_Stocks/*.md; do
  grep "^data_source:" "$f" | sed 's/data_source: *//'
done | sort | uniq -c
```

Expected (spec §7 기준):
- yahoo ≥ 8
- investing ≤ 3
- websearch ≤ 1
- failed ≤ 1
- holiday는 휴장일에만

- [ ] **Step 4: 12종목 frontmatter `업데이트` 필드 검증**

```bash
TODAY=$(date +%Y-%m-%d)
echo "오늘: $TODAY"
echo ""
for f in 20_Stocks/*.md; do
  upd=$(grep "^업데이트:" "$f" | sed 's/업데이트: *//')
  ds=$(grep "^data_source:" "$f" | sed 's/data_source: *//')
  if [ "$upd" = "$TODAY" ]; then
    echo "✅ $(basename "$f" .md): $upd (source: $ds)"
  else
    echo "⚠️  $(basename "$f" .md): $upd (source: $ds, 오늘 아님)"
  fi
done
```

Expected (휴장 아닌 경우):
- ✅ 11개 이상 (12개 중 1개 정도 failed 허용)
- failed 종목은 ⚠️로 표시되며 data_source=failed

- [ ] **Step 5: Dashboard.md 검증**

```bash
RAW=$(cat "10_Summary/Dashboard.md")

# 마지막 갱신 시각이 오늘인지
echo "$RAW" | grep -m1 "마지막 갱신:"

# §2 표에 ⚠️ 자동 부착 패턴이 있는지 (failed 종목만)
echo ""
echo "=== ⚠️ 자동 부착 종목 ==="
echo "$RAW" | grep "⚠️" | head -20

# 경고 섹션에 cascade-specific 메시지가 있는지
echo ""
echo "=== 경고 섹션 ==="
echo "$RAW" | sed -n '/## ⚠️ 경고/,/^---/p'
```

Expected:
- 마지막 갱신 시각 = 오늘 21:0X KST
- ⚠️ 부착 종목 = failed 종목과 일치 (없으면 경고 섹션도 깨끗)
- 경고 섹션에 `fallback 소스 사용` 또는 `모든 소스 실패` 또는 `없음` 중 하나만 표시

기존 Dashboard의 "Naver Finance 데이터 갱신 실패" 같은 사람이 적은 경고는 사라져야 함 (routine이 자동 생성한 새 패턴만 존재).

- [ ] **Step 6: spec §7 임계 평가**

Step 3 결과를 다음 기준으로 평가:

| 케이스 | 판정 | 후속 |
|--------|------|------|
| yahoo ≥ 8 | ✅ Primary 정상 | 진행 |
| yahoo 4~7 | ⚠️ Primary 불안 | 1주 모니터링 |
| yahoo ≤ 3 | ❌ Primary 부적합 | cascade 순서 재조정 검토, brainstorming 재개 |
| failed ≥ 3 | ❌ cascade 부적합 | Fallback 소스 재검토, brainstorming 재개 |

✅인 경우 Step 7로. ⚠️/❌인 경우 사용자에게 보고하고 spec 수정 결정.

- [ ] **Step 7: 작업 임시 파일 정리 + plan 완료 commit**

```bash
rm -rf docs/_workspace/
git add -A
git commit -m "$(cat <<'EOF'
feat: 데이터 fetch cascade routine 적용 완료

routine trig_01SZu6hi1mTygQ7G6ztER4Ch prompt 업데이트:
- Step A를 Yahoo → Investing → WebSearch 3단계 cascade로 교체
- frontmatter에 data_source 필드 신규 기록
- Dashboard §2 표 ⚠️ 자동 부착 + cascade-specific 경고

1회 수동 실행 검증 결과:
- 12종목 data_source 분포: yahoo X / investing Y / websearch Z / failed W
- 업데이트 날짜 ✅ N개 / ⚠️ M개

작업 임시 파일 (docs/_workspace/) 정리.

Plan: docs/superpowers/plans/2026-05-28-data-fetch-reliability.md
Spec: docs/superpowers/specs/2026-05-28-data-fetch-reliability-design.md
EOF
)"
```

`X/Y/Z/W/N/M`은 실제 Step 3, 4 결과로 치환.

- [ ] **Step 8: push**

```bash
git push origin main
```

이 task가 plan의 최종 단계.

---

## Self-Review Checklist (작성 후 확인)

- [x] **Spec coverage**:
  - §3 cascade → Task 3 Step 2 (Step A 교체)
  - §4 필드 매핑 → Task 3 Step 2~5 (각 필드의 source 우선순위)
  - §5 frontmatter data_source → Task 3 Step 4 (Step B 수정)
  - §5 Dashboard 경고 + ⚠️ 자동 → Task 3 Step 5 (Step D 수정)
  - §6 routine prompt 변경 → Task 3, 4
  - §7 검증 → Task 5 Step 3~6
  - §8 위험 완화 → 각 task의 fallback step (Task 2 Step 4, Task 4 Step 4, Task 5 Step 6)
  - §9 후속 spec → plan 외 (별도 brainstorming)
- [x] **Placeholder scan**: TBD/TODO/"implement later" 없음. 모든 step에 명령/검증 기준/실패 시 행동 포함.
- [x] **Type consistency**:
  - `data_source` 값 enum (`yahoo`/`investing`/`websearch`/`failed`/`holiday`) — spec과 plan 일치
  - routine ID `trig_01SZu6hi1mTygQ7G6ztER4Ch` — 모든 task 동일
  - 파일 경로 `docs/_workspace/2026-05-28-*` — 모든 task 동일
  - cascade 단계 명명 `A.1/A.2/A.3/A.4/A.5/A.6` — Task 3 내 일관
- [x] **Phase 분리**: 이번 plan은 데이터 fetch만. 매매일지 UX는 별도 spec/plan.

---

## Notes

- Task 1, 2는 작업 임시 파일에만 영향 — 실수해도 routine은 안 망가짐
- Task 4가 핵심 리스크 (cloud routine prompt 교체). Task 1 백업이 있으므로 망가지면 백업 prompt로 복원 명령 실행
- Task 5 Step 2의 `sleep 600`은 cloud 환경 latency 고려. 더 빨리 끝나는 경우도 있으므로 5분 후 한 번 확인하고 안 끝났으면 추가 대기도 가능
- 휴장일에 Task 5를 실행하면 가격 갱신 안 됨이 정상 — `data_source=holiday` 표시되면 검증 통과
- 1주 누적 모니터링은 plan 외부 운용 사이클로 처리 (spec §7 후반부)
