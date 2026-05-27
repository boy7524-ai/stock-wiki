# Stock Wiki — 매매일지 + Dashboard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** stock-wiki에 매매일지(40_Trades/매매일지.md) + 종목 비교 Dashboard(10_Summary/Dashboard.md) 기능을 추가하고, 21시 클라우드 routine이 매일 자동 집계·갱신하도록 prompt를 업데이트한다.

**Architecture:** 새 마크다운 파일 2개 생성 + 기존 12종목 노트에 YAML frontmatter prepend. 이후 cloud routine `trig_01SZu6hi1mTygQ7G6ztER4Ch`의 prompt에 4단계(가격 fetch → frontmatter 갱신 → 매매일지 파싱 → Dashboard 재생성)를 추가한다. 사용자는 매매일지 거래 row 추가가 유일한 수동 작업.

**Tech Stack:** Markdown + YAML frontmatter, Git, Claude Code cron routine (sonnet-4-6), Naver Finance + Yahoo Finance Japan (가격 fetch via WebFetch).

**Spec 참조:** `docs/superpowers/specs/2026-05-28-stock-wiki-trading-journal-dashboard-design.md`

---

## File Structure

| 파일 | 책임 | 누가 편집 |
|------|------|----------|
| `40_Trades/매매일지.md` | 거래 내역(사용자 입력 표) + 보유 포지션·실현손익 섹션(routine 자동) | 사용자: "거래 내역" 표만 / Routine: 나머지 |
| `10_Summary/Dashboard.md` | 포트폴리오 요약 + §1 포지션 + §2 펀더멘털 두 개 표 | Routine 단독 (사용자 편집 금지) |
| `20_Stocks/*.md` (12 files) | 본문 위에 YAML frontmatter 추가 — 가격·PER·목표가·리스크 메타데이터 | Routine: frontmatter만 / 사용자: 본문만 |
| Cloud routine `trig_01SZu6hi1mTygQ7G6ztER4Ch` | Prompt에 4개 신규 단계 추가 (이 repo의 파일 변경 아님) | 한 번 업데이트 후 매일 21시 자동 실행 |

---

## Task 1: 40_Trades/매매일지.md 생성

**Files:**
- Create: `40_Trades/매매일지.md`

- [ ] **Step 1: 파일 작성**

Write 도구로 `40_Trades/매매일지.md`에 다음을 정확히 저장:

```markdown
# 매매일지

> 매수/매도 발생 시 아래 "거래 내역" 표 맨 아래에 한 줄 추가하면 됩니다.
> 나머지 섹션(보유 포지션, 실현손익 누계)은 매일 21시 routine이 자동 갱신합니다.

## 거래 내역
| 날짜 | 종목 | 구분 | 수량 | 단가 | 수수료 | 메모 |
|------|------|------|------|------|--------|------|

<!-- 아래 섹션은 routine 자동 생성 — 직접 편집 금지 -->

## 보유 포지션 (routine 자동 갱신)

_거래 입력 후 routine이 실행되면 자동으로 채워집니다._

| 종목 | 보유수량 | 평단가 | 매수금액 | 현재가 | 평가금액 | 미실현 손익 | 미실현 수익률 |
|------|---------|--------|----------|--------|----------|------------|--------------|

## 실현손익 누계 (routine 자동 갱신)

_매도 거래가 발생하고 routine이 실행되면 자동으로 채워집니다._

| 종목 | 실현손익 | 매도 횟수 |
|------|---------|----------|
```

- [ ] **Step 2: 검증**

```powershell
$path = "40_Trades/매매일지.md"
$raw = Get-Content $path -Raw
$checks = @(
    @{ name = "파일 존재"; ok = Test-Path $path },
    @{ name = "거래 내역 헤더"; ok = $raw -match "## 거래 내역" },
    @{ name = "보유 포지션 섹션"; ok = $raw -match "## 보유 포지션" },
    @{ name = "실현손익 섹션"; ok = $raw -match "## 실현손익 누계" },
    @{ name = "거래 표 컬럼 7개"; ok = $raw -match "\| 날짜 \| 종목 \| 구분 \| 수량 \| 단가 \| 수수료 \| 메모 \|" }
)
$checks | ForEach-Object { if ($_.ok) { "✅ $($_.name)" } else { "❌ $($_.name)" } }
```

Expected: 5개 모두 ✅

- [ ] **Step 3: Commit**

```bash
git add 40_Trades/매매일지.md
git commit -m "feat: 40_Trades/매매일지.md 초기 템플릿 추가"
```

---

## Task 2: 10_Summary/Dashboard.md placeholder 생성

**Files:**
- Create: `10_Summary/Dashboard.md`

- [ ] **Step 1: 파일 작성**

Write 도구로 `10_Summary/Dashboard.md`에 다음을 정확히 저장:

```markdown
# 종목 Dashboard

_첫 21시 routine 실행 후 자동으로 채워집니다. 직접 편집하지 마세요 — routine이 매일 재생성합니다._

## 📊 포트폴리오 요약

(routine 실행 대기)

## 💼 §1 내 포지션

(routine 실행 대기)

## 🔍 §2 종목 펀더멘털

(routine 실행 대기)

## ⚠️ 경고

(routine 실행 대기)
```

- [ ] **Step 2: 검증**

```powershell
$path = "10_Summary/Dashboard.md"
Test-Path $path
```
Expected: `True`

- [ ] **Step 3: Commit**

```bash
git add 10_Summary/Dashboard.md
git commit -m "feat: 10_Summary/Dashboard.md placeholder 추가"
```

---

## Task 3: 12종목 frontmatter 추가 (Phase 1 핵심 필드만)

**Files (12 modify):**
`20_Stocks/SK하이닉스.md`, `20_Stocks/HD현대중공업.md`, `20_Stocks/삼성바이오로직스.md`, `20_Stocks/한화에어로스페이스.md`, `20_Stocks/메리츠금융지주.md`, `20_Stocks/대한항공.md`, `20_Stocks/코웨이.md`, `20_Stocks/풍산.md`, `20_Stocks/동성화인텍.md`, `20_Stocks/케이엠더블유.md`, `20_Stocks/실리콘투.md`, `20_Stocks/고베물산.md`

### 메타데이터 매핑 (Step 1에서 사용)

| 종목 | 종목코드 | 시장 | 섹터 | 통화 |
|------|---------|------|------|------|
| SK하이닉스 | 000660 | KOSPI | 반도체 | KRW |
| HD현대중공업 | 329180 | KOSPI | 조선·중공업 | KRW |
| 삼성바이오로직스 | 207940 | KOSPI | 바이오 | KRW |
| 한화에어로스페이스 | 012450 | KOSPI | 방산 | KRW |
| 메리츠금융지주 | 138040 | KOSPI | 금융 | KRW |
| 대한항공 | 003490 | KOSPI | 항공 | KRW |
| 코웨이 | 021240 | KOSPI | 가전·렌탈 | KRW |
| 풍산 | 103140 | KOSPI | 비철금속·방산 | KRW |
| 동성화인텍 | 033500 | KOSDAQ | LNG 보냉재 | KRW |
| 케이엠더블유 | 032500 | KOSDAQ | 통신장비 | KRW |
| 실리콘투 | 257720 | KOSDAQ | K-뷰티 유통 | KRW |
| 고베물산 | 3038 | 일본 | 식품 유통 | JPY |

### Frontmatter 템플릿 (각 파일에 prepend할 블록)

`<CODE>`, `<MARKET>`, `<SECTOR>`, `<CURRENCY>`를 매핑 표 값으로 치환해 각 종목별로 사용:

```yaml
---
종목코드: "<CODE>"
섹터: <SECTOR>
시장: <MARKET>
통화: <CURRENCY>

# 가격 (매일 routine 갱신)
현재가:
전일대비:
업데이트:

# 밸류에이션 (매일 routine 갱신)
PER:
PER_forward:
시가총액:

# 펀더멘털 (실적 발표 트리거 시 갱신)
영업이익증가율:

# 목표가 (컨센서스 변동 시 갱신)
목표가_컨센서스:
목표가_괴리율:

# 리스크 플래그
리스크: []
---

```

(블록 끝에 빈 줄 한 줄 포함 — 본문과 분리)

- [ ] **Step 1: 각 종목 파일에 frontmatter prepend**

12종목 각각에 대해:
1. Read 도구로 파일 첫 줄을 확인 (보통 `# 종목명`)
2. Edit 도구로 첫 줄 앞에 frontmatter 블록 삽입 (매핑 표 값 치환)

예시 — SK하이닉스의 경우:

```
old_string: "# SK하이닉스"
new_string: """---
종목코드: "000660"
섹터: 반도체
시장: KOSPI
통화: KRW

# 가격 (매일 routine 갱신)
현재가:
전일대비:
업데이트:

# 밸류에이션 (매일 routine 갱신)
PER:
PER_forward:
시가총액:

# 펀더멘털 (실적 발표 트리거 시 갱신)
영업이익증가율:

# 목표가 (컨센서스 변동 시 갱신)
목표가_컨센서스:
목표가_괴리율:

# 리스크 플래그
리스크: []
---

# SK하이닉스"""
```

12개 파일 동일 패턴, 메타데이터 값만 다름. (Edit `replace_all=false`)

- [ ] **Step 2: YAML 구조 검증**

```powershell
$files = Get-ChildItem "20_Stocks/*.md"
$failed = @()
foreach ($f in $files) {
    $raw = Get-Content $f.FullName -Raw
    if (-not $raw.StartsWith("---")) {
        $failed += "$($f.Name): frontmatter 시작 누락"
        continue
    }
    $yaml = ($raw -split "---", 3)[1]
    $required = @("종목코드:", "섹터:", "시장:", "통화:", "현재가:", "PER:", "PER_forward:", "리스크:")
    $missing = $required | Where-Object { $yaml -notmatch [regex]::Escape($_) }
    if ($missing) {
        $failed += "$($f.Name): 누락 키 $($missing -join ', ')"
    } else {
        Write-Output "✅ $($f.Name)"
    }
}
if ($failed) {
    Write-Output "---"
    $failed | ForEach-Object { Write-Output "❌ $_" }
}
```

Expected: 12개 모두 ✅, 실패 없음

- [ ] **Step 3: 종목코드/섹터 사용자 검수 (옵션)**

매핑 표의 종목코드가 정확한지 사용자가 확인. 1개라도 틀리면 해당 파일 Edit 수정 후 Step 2 재실행.

- [ ] **Step 4: Commit**

```bash
git add 20_Stocks/*.md
git commit -m "feat: 12종목 노트에 frontmatter 추가 (가격·밸류·리스크 필드 스키마)"
```

---

## Task 4: Cloud routine prompt에 4단계 추가

routine ID: `trig_01SZu6hi1mTygQ7G6ztER4Ch`

이 task는 repo 파일 변경 없음 — claude.ai 측 routine config 업데이트.

- [ ] **Step 1: 현재 routine prompt 조회**

```bash
claude -p "routine ID trig_01SZu6hi1mTygQ7G6ztER4Ch의 현재 prompt 전문을 그대로 출력해줘. 절대 요약하지 말고 원문 그대로."
```

출력된 prompt 전문을 파일에 임시 저장 (`/tmp/routine-current.txt` 또는 메모리):

```powershell
claude -p "routine ID trig_01SZu6hi1mTygQ7G6ztER4Ch의 현재 prompt 전문을 그대로 출력해줘. 절대 요약하지 말고 원문 그대로." > $env:TEMP\routine-current.txt
Get-Content $env:TEMP\routine-current.txt
```

- [ ] **Step 2: 새 prompt 작성**

기존 prompt에서 "1. 12종목 뉴스 검색" 단계 직후, "일일 브리핑 생성" 단계 직전 위치에 다음 4단계 블록을 삽입:

````text

## 신규 단계: 가격·메타데이터 갱신 + Dashboard 생성

### Step A: 12종목 가격·밸류에이션 fetch

각 종목별로 다음 데이터 수집:
- 한국 종목 (KOSPI/KOSDAQ): Naver Finance 페이지 (https://finance.naver.com/item/main.naver?code=<종목코드>)에서 WebFetch로 다음 항목 추출:
  - 현재가 (당일 종가)
  - 전일대비 (%)
  - PER (Trailing 12M)
  - PER_forward (Forward 12M, 추정 PER로 표기된 값)
  - 시가총액 (억원 단위)
  - 영업이익증가율 (최근 분기 YoY)
  - 목표가 컨센서스 (증권사 평균)
- 일본 종목 (고베물산, 종목코드 3038): Yahoo Finance Japan (https://finance.yahoo.co.jp/quote/3038.T)에서 동일 항목.
  현재가는 JPY 종가 → 당일 KRW/JPY 환율로 환산해 KRW로 저장.
- fetch 실패 시 해당 종목의 frontmatter 업데이트 필드는 갱신하지 말고 (이전 거래일 값 유지), Dashboard 경고 섹션에 `📡 [종목] 데이터 갱신 실패` 기록.

### Step B: 20_Stocks/<종목>.md frontmatter merge update

각 종목 노트의 frontmatter 블록(--- ~ ---) 안의 필드만 갱신, 본문은 절대 건드리지 말 것.

- 매일 무조건 갱신: 현재가, 전일대비, 업데이트(오늘 날짜 YYYY-MM-DD), 목표가_괴리율 (= (목표가_컨센서스 - 현재가) / 현재가 × 100)
- fetch 값이 이전과 다를 때만 갱신: PER, PER_forward, 시가총액, 영업이익증가율, 목표가_컨센서스, 리스크
- 절대 덮어쓰지 말 것 (사용자 검수 필드): 종목코드, 섹터, 시장, 통화

PER을 노출하는 모든 출력(브리핑·dashboard 등)에서 Trailing PER과 Forward PER을 항상 함께 표기 ("12.3 / 9.8" 형식).

### Step C: 40_Trades/매매일지.md 파싱 + derived 계산

매매일지 파일의 "## 거래 내역" 표만 읽고 (다른 섹션은 무시):
- 종목별로 보유수량 = Σ 매수수량 - Σ 매도수량
- 평단가 = 가중평균 매수단가 (수수료 제외; 매수: 새 평단가 = (이전 보유금액 + 신규 매수금액) / 새 총수량; 매도: 평단가 유지)
- 매수금액 = 보유수량 × 평단가
- 평가금액 = 보유수량 × 현재가 (frontmatter에서)
- 미실현 손익 = 평가금액 - 매수금액
- 미실현 수익률 = (현재가 - 평단가) / 평단가 × 100
- 비중 = 평가금액 / Σ 모든 평가금액 × 100
- 실현손익 = Σ (매도단가 - 평단가) × 매도수량 - 매도수수료
- 매도 횟수 = 종목별 매도 row 개수

파싱 오류 row (수량 비숫자, 종목 wiki-link 깨짐 등)는 무시하고 경고 섹션에 row 번호+사유 기록.

이후 40_Trades/매매일지.md의 "## 보유 포지션 (routine 자동 갱신)" 섹션과 "## 실현손익 누계 (routine 자동 갱신)" 섹션의 표를 다시 작성 (사용자가 편집하는 "거래 내역" 표는 절대 건드리지 말 것).

### Step D: 10_Summary/Dashboard.md 전체 재생성

전체 파일을 새 내용으로 덮어쓰기. 구조:

```
# 종목 Dashboard
*마지막 갱신: YYYY-MM-DD HH:MM KST*

## 📊 포트폴리오 요약
- 총 매수금액: N 원
- 총 평가금액: N 원
- **미실현 손익: +N 원 (+N.NN%)**
- 실현손익 누계: +N 원
- 보유 종목 수: N / 추적 종목 수: 12

## 💼 §1 내 포지션
| 종목 | 보유수량 | 평단가 | 현재가 | 평가금액 | 미실현 손익 | 수익률 | 비중 | 실현손익 |
(비중 desc 정렬, 합계 row 굵게)

## 🔍 §2 종목 펀더멘털 (12종목 전체)
| 종목 | 섹터 | 현재가 | 전일대비 | PER (T/F) | 영업이익↑ | 목표가 | 괴리율 | 리스크 |
(보유종목 먼저 → 미보유 종목 → 섹터 알파순; PER은 "Trailing/Forward" 두 값 모두 표기)

## ⚠️ 경고
- (보유수량 부족 매도, 신규 종목, fetch 실패, 매매일지 파싱 오류 모두 표시. 없으면 "없음")

---
*Powered by 21시 routine · 데이터 출처: Naver Finance / Yahoo Finance Japan*
```

수익률 ≥ +10%면 굵게, ≤ -10%면 굵게 + 🔻.

매매일지에 거래가 하나도 없으면 §1 포지션 표는 빈 표 + "보유 포지션 없음" 텍스트; 포트폴리오 요약은 모두 0; §2 펀더멘털 표는 정상 출력.

````

### Step E: commit & push 단계에 Dashboard.md·매매일지.md·20_Stocks 변경 모두 포함

기존 단계 그대로.

- [ ] **Step 3: routine prompt 업데이트 실행**

위 Step 1에서 받은 기존 prompt와 Step 2의 4단계 블록을 합쳐 새 prompt 완성. 다음 명령으로 업데이트:

```bash
claude -p "routine ID trig_01SZu6hi1mTygQ7G6ztER4Ch의 prompt를 다음 내용으로 업데이트해줘. 다른 설정(스케줄·도구·모델)은 그대로 유지. 새 prompt: <새 prompt 전문>"
```

또는 schedule 스킬을 invoke해서 "routine X의 prompt를 다음으로 업데이트" 요청.

- [ ] **Step 4: 업데이트 검증**

```bash
claude -p "routine ID trig_01SZu6hi1mTygQ7G6ztER4Ch의 현재 prompt에 'Step A: 12종목 가격', '20_Stocks/<종목>.md frontmatter merge update', '10_Summary/Dashboard.md', '40_Trades/매매일지.md' 네 개 문구가 모두 들어있는지 yes/no로 답해줘"
```

Expected: 네 개 모두 yes.

```bash
claude routines list
```

Expected: 동일 routine ID, enabled, 다음 실행 시간 정상(매일 21:01 KST).

이 task는 commit 없음 (cloud config 변경, repo 변경 없음).

---

## Task 5: Routine 1회 수동 실행 + 결과 verification

- [ ] **Step 1: routine 즉시 1회 실행**

```bash
claude -p "routine ID trig_01SZu6hi1mTygQ7G6ztER4Ch을 지금 한 번 실행해줘. 완료될 때까지 기다리지 말고 시작만 시켜줘."
```

실행 시작 후 약 5분 소요 예상 (12종목 가격 fetch + frontmatter 갱신 + dashboard 생성).

- [ ] **Step 2: 실행 완료 후 변경사항 pull**

routine은 cloud에서 commit & push 하므로 로컬은 pull로 결과 수신.

```bash
git pull --rebase --autostash
git log --oneline -3
```

Expected: routine이 만든 새 commit(예: `daily: 2026-MM-DD 브리핑 및 종목 업데이트`)이 보임.

- [ ] **Step 3: Dashboard.md 검증**

```powershell
$path = "10_Summary/Dashboard.md"
$raw = Get-Content $path -Raw
$checks = @(
    @{ name = "마지막 갱신 시각"; ok = $raw -match "마지막 갱신:" },
    @{ name = "포트폴리오 요약 섹션"; ok = $raw -match "## 📊 포트폴리오 요약" },
    @{ name = "§1 내 포지션 표 헤더"; ok = $raw -match "보유수량 \| 평단가 \| 현재가" },
    @{ name = "§2 펀더멘털 표 헤더"; ok = $raw -match "PER \(T/F\)" },
    @{ name = "12종목 모두 등장"; ok = (@("SK하이닉스","HD현대중공업","삼성바이오로직스","한화에어로스페이스","메리츠금융지주","대한항공","코웨이","풍산","동성화인텍","케이엠더블유","실리콘투","고베물산") | Where-Object { $raw -notmatch $_ }).Count -eq 0 }
)
$checks | ForEach-Object { if ($_.ok) { "✅ $($_.name)" } else { "❌ $($_.name)" } }
```

Expected: 5개 모두 ✅

- [ ] **Step 4: 20_Stocks frontmatter 갱신 검증**

```powershell
$files = Get-ChildItem "20_Stocks/*.md"
$today = Get-Date -Format "yyyy-MM-dd"
$failed = @()
foreach ($f in $files) {
    $raw = Get-Content $f.FullName -Raw
    if ($raw -notmatch "현재가:\s*\d") { $failed += "$($f.Name): 현재가 비어있음" }
    elseif ($raw -notmatch "PER:\s*\d") { $failed += "$($f.Name): PER 비어있음" }
    elseif ($raw -notmatch "PER_forward:\s*\d") { $failed += "$($f.Name): PER_forward 비어있음" }
    elseif ($raw -notmatch "업데이트:\s*$today") { $failed += "$($f.Name): 업데이트 날짜 != $today (휴장일 가능, 경고만)" }
    else { Write-Output "✅ $($f.Name)" }
}
$failed | ForEach-Object { Write-Output "⚠️  $_" }
```

Expected: 12개 모두 ✅. 휴장일이면 "업데이트 날짜 != $today" 경고 허용 (Dashboard 경고 섹션에 휴장 표시 있는지 같이 확인).

- [ ] **Step 5: 매매일지 보유 포지션 섹션 검증**

```powershell
$raw = Get-Content "40_Trades/매매일지.md" -Raw
if ($raw -match "## 보유 포지션" -and $raw -match "## 실현손익 누계" -and $raw -match "## 거래 내역") {
    Write-Output "✅ 매매일지 3개 섹션 모두 유지"
} else {
    Write-Output "❌ 매매일지 구조 손상"
}
```

거래 내역에 row가 없으면 보유 포지션·실현손익 누계 표는 빈 표 OK.

- [ ] **Step 6: 실패 시 디버깅 가이드**

검증 실패 케이스별:
- **Dashboard 일부 종목 누락**: routine이 일부 종목 fetch 실패 → Dashboard 경고 섹션 확인, 문제 종목 코드 재확인 후 prompt에서 fetch URL 수정
- **frontmatter 비어있음**: routine prompt의 Step B(frontmatter merge update) 동작 실패 → claude.ai/code/routines/trig_01SZu6hi1mTygQ7G6ztER4Ch 로그 확인
- **매매일지 구조 손상**: routine이 "거래 내역" 표를 건드림 → prompt에 "거래 내역 표는 절대 건드리지 말 것" 강조 보강
- **PER_forward 누락**: Naver Finance에서 "추정 PER" 항목 추출 실패 → prompt에서 대체 소스(증권사 컨센서스 페이지) 추가

수정 후 Task 4 Step 3 재실행 → Task 5 Step 1부터 재검증.

이 task는 commit 없음 (routine이 이미 cloud에서 commit & push 했음).

---

## Self-Review Checklist (작성 후 확인)

- [x] **Spec coverage**: §1~§10 모든 섹션이 Task 1~5에 매핑됨. §10 비목표는 의도적 제외.
- [x] **Placeholder scan**: TBD/TODO/"implement later" 없음. 모든 step에 구체적 code/command 포함.
- [x] **Type consistency**: 필드명 `종목코드`/`섹터`/`시장`/`통화`/`현재가`/`PER`/`PER_forward`/`목표가_컨센서스`/`목표가_괴리율`/`리스크` 모든 Task에서 동일하게 사용.
- [x] **Phase 4 (후속)** 작업은 의도적으로 plan 제외 — 별도 spec.

---

## Notes

- Task 1, 2, 3은 mechanical → 빠르게 진행 가능
- Task 4가 핵심 리스크 (cloud routine prompt 변경) → 신중하게 prompt 작성, Step 1에서 기존 prompt 백업 필수
- Task 5 verification에서 부분 실패는 흔함 — 정상 종목 11개 + fetch 실패 1개라면 prompt 일부 수정으로 해결 가능
- 휴장일에 Task 5를 실행하면 `현재가`가 채워지지 않을 수 있음 — 비휴장 거래일에 검증 권장
