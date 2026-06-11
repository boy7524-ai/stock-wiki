---
title: Obsidian 볼트 설정 및 종목 관리
date: 2026-06-11
status: approved
---

## 개요

현재 `TEST/` 루트 디렉토리를 Obsidian 볼트로 설정하여 전체 주식 위키 자료(`20_Stocks/`, `30_Daily_Log/`, `40_Trades/`, `10_Summary/`)를 Obsidian에서 열람 가능하게 한다. 종목 추가/삭제는 Claude Code 세션에서 Claude에게 요청하는 방식으로 처리한다.

## 현재 상태

- `stock_wiki/.obsidian/` — Obsidian 볼트 설정 파일 존재 (Bases 기능 활성화됨)
- `stock_wiki/무제/무제.base` — 빈 Bases 파일
- `20_Stocks/` — 12개 종목 MD 파일 (YAML frontmatter 포함)
- `30_Daily_Log/`, `40_Trades/`, `10_Summary/` — 기타 자료

## 설계

### 1. 볼트 루트 변경

`stock_wiki/.obsidian/`을 `TEST/.obsidian/`으로 이동한다. 이후 `stock_wiki/` 폴더는 제거한다.

파일 경로 변경 없음 — 기존 자동화 스크립트(일일 브리핑, 종목 업데이트) 영향 없음.

### 2. 제외 폴더 설정

Obsidian `app.json`에 `userIgnoreFilters` 추가:
- `docs/` — 설계/계획 문서 (Obsidian에서 불필요)
- `.claude/` — Claude Code 내부 파일

### 3. Bases 종목 테이블 뷰

`10_Summary/종목_테이블.base` 생성. `20_Stocks/` 전체 파일을 아래 컬럼으로 표시:

| 컬럼 | YAML 속성 |
|------|-----------|
| 종목명 | file.name |
| 종목코드 | 종목코드 |
| 섹터 | 섹터 |
| 시장 | 시장 |
| 현재가 | 현재가 |
| PER | PER |
| PER_forward | PER_forward |
| 목표가 | 목표가_컨센서스 |
| 목표가_괴리율 | 목표가_괴리율 |
| 업데이트 | 업데이트 |

### 4. 종목 추가

트리거: Claude Code 세션에서 "XX 종목 추가해줘" 요청

동작:
1. 웹 검색으로 최신 데이터 수집 (현재가, PER, 목표가 컨센서스, 섹터, 최근 이슈 등)
2. 기존 종목 파일 형식(YAML frontmatter + 핵심 요약 / 최근 실적 / 최신 이슈 섹션)으로 `20_Stocks/<종목명>.md` 생성

### 5. 종목 삭제

트리거: Claude Code 세션에서 "XX 종목 삭제해줘" 요청

동작:
1. `20_Stocks/<종목명>.md` 삭제
2. 해당 종목이 최근 30일 브리핑(`30_Daily_Log/`)에 포함된 경우 사용자에게 알림

## 제약 사항

- 종목 추가/삭제는 Claude Code 세션에서만 파일에 즉시 반영됨 (claude.ai 채팅 불가)
- Obsidian Bases 기능은 Obsidian 1.8+ 필요
