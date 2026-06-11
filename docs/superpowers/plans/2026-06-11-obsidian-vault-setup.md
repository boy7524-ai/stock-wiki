# Obsidian 볼트 설정 및 종목 관리 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `TEST/` 루트 디렉토리를 Obsidian 볼트로 설정하여 전체 주식 위키 자료를 Obsidian에서 열람 가능하게 하고, 종목 테이블 Bases 뷰를 구성한다.

**Architecture:** `stock_wiki/.obsidian/`을 `TEST/.obsidian/`으로 이동하여 볼트 루트를 변경한다. `docs/`와 `.claude/` 폴더는 excluded 처리한다. `10_Summary/종목_테이블.base`에 Bases 뷰를 설정하고 `stock_wiki/` 폴더를 삭제한다.

**Tech Stack:** Obsidian 1.8+, PowerShell (Windows), JSON 설정 파일

---

### Task 1: `.obsidian/` 디렉토리 이동

**Files:**
- Create: `TEST/.obsidian/app.json`
- Create: `TEST/.obsidian/core-plugins.json`
- Create: `TEST/.obsidian/appearance.json`
- Create: `TEST/.obsidian/graph.json`

- [ ] **Step 1: TEST/.obsidian/ 디렉토리 생성 및 파일 복사**

PowerShell에서 실행:
```powershell
New-Item -ItemType Directory -Force "C:\Users\김민철\Documents\TEST\.obsidian"
Copy-Item "C:\Users\김민철\Documents\TEST\stock_wiki\.obsidian\core-plugins.json" "C:\Users\김민철\Documents\TEST\.obsidian\core-plugins.json"
Copy-Item "C:\Users\김민철\Documents\TEST\stock_wiki\.obsidian\appearance.json" "C:\Users\김민철\Documents\TEST\.obsidian\appearance.json"
Copy-Item "C:\Users\김민철\Documents\TEST\stock_wiki\.obsidian\graph.json" "C:\Users\김민철\Documents\TEST\.obsidian\graph.json"
```

Expected: `.obsidian/` 디렉토리와 파일 3개 생성됨

- [ ] **Step 2: 복사 확인**

```powershell
Get-ChildItem "C:\Users\김민철\Documents\TEST\.obsidian"
```

Expected: `appearance.json`, `core-plugins.json`, `graph.json` 3개 파일 출력

---

### Task 2: `app.json` 생성 (excluded 폴더 설정 포함)

**Files:**
- Create: `TEST/.obsidian/app.json`

- [ ] **Step 1: app.json 작성**

`TEST/.obsidian/app.json` 파일을 아래 내용으로 생성:

```json
{
  "userIgnoreFilters": [
    "docs",
    ".claude"
  ]
}
```

- [ ] **Step 2: 내용 확인**

```powershell
Get-Content "C:\Users\김민철\Documents\TEST\.obsidian\app.json"
```

Expected: `userIgnoreFilters` 배열에 `"docs"`, `".claude"` 포함된 JSON 출력

---

### Task 3: `10_Summary/종목_테이블.base` 생성

**Files:**
- Create: `TEST/10_Summary/종목_테이블.base`

- [ ] **Step 1: Bases 파일 생성**

`TEST/10_Summary/종목_테이블.base` 파일을 아래 내용으로 생성:

```json
{
  "filters": [
    {
      "id": "folder-filter",
      "field": "file.folder",
      "operator": "equals",
      "value": "20_Stocks"
    }
  ],
  "views": [
    {
      "id": "table-view",
      "name": "종목 테이블",
      "type": "table",
      "columns": {
        "file": { "width": 180 },
        "종목코드": { "width": 100 },
        "섹터": { "width": 180 },
        "시장": { "width": 80 },
        "현재가": { "width": 100 },
        "PER": { "width": 80 },
        "PER_forward": { "width": 100 },
        "목표가_컨센서스": { "width": 130 },
        "목표가_괴리율": { "width": 110 },
        "업데이트": { "width": 110 }
      },
      "order": [
        "file",
        "종목코드",
        "섹터",
        "시장",
        "현재가",
        "PER",
        "PER_forward",
        "목표가_컨센서스",
        "목표가_괴리율",
        "업데이트"
      ],
      "filters": [],
      "sort": []
    }
  ]
}
```

- [ ] **Step 2: 파일 존재 확인**

```powershell
Test-Path "C:\Users\김민철\Documents\TEST\10_Summary\종목_테이블.base"
```

Expected: `True`

---

### Task 4: `workspace.json` 업데이트

**Files:**
- Create: `TEST/.obsidian/workspace.json`

현재 `workspace.json`은 `무제/무제.base`를 참조하고 있음. 새 볼트 루트 기준으로 `10_Summary/종목_테이블.base`를 열도록 업데이트한다.

- [ ] **Step 1: workspace.json 작성**

`TEST/.obsidian/workspace.json` 파일을 아래 내용으로 생성:

```json
{
  "main": {
    "id": "main-split",
    "type": "split",
    "children": [
      {
        "id": "main-tabs",
        "type": "tabs",
        "children": [
          {
            "id": "bases-leaf",
            "type": "leaf",
            "state": {
              "type": "bases",
              "state": {
                "file": "10_Summary/종목_테이블.base",
                "viewName": "종목 테이블"
              },
              "icon": "lucide-table",
              "title": "종목_테이블"
            }
          }
        ]
      }
    ],
    "direction": "vertical"
  },
  "left": {
    "id": "left-split",
    "type": "split",
    "children": [
      {
        "id": "left-tabs",
        "type": "tabs",
        "children": [
          {
            "id": "file-explorer-leaf",
            "type": "leaf",
            "state": {
              "type": "file-explorer",
              "state": {
                "sortOrder": "alphabetical",
                "autoReveal": false
              },
              "icon": "lucide-folder-closed",
              "title": "파일 탐색기"
            }
          },
          {
            "id": "search-leaf",
            "type": "leaf",
            "state": {
              "type": "search",
              "state": {
                "query": "",
                "matchingCase": false,
                "explainSearch": false,
                "collapseAll": false,
                "extraContext": false,
                "sortOrder": "alphabetical"
              },
              "icon": "lucide-search",
              "title": "검색"
            }
          }
        ]
      }
    ],
    "direction": "horizontal",
    "width": 300
  },
  "right": {
    "id": "right-split",
    "type": "split",
    "children": [
      {
        "id": "right-tabs",
        "type": "tabs",
        "children": [
          {
            "id": "tag-leaf",
            "type": "leaf",
            "state": {
              "type": "tag",
              "state": {
                "sortOrder": "frequency",
                "useHierarchy": true,
                "showSearch": false,
                "searchQuery": ""
              },
              "icon": "lucide-tags",
              "title": "태그"
            }
          }
        ]
      }
    ],
    "direction": "horizontal",
    "width": 300,
    "collapsed": true
  },
  "left-ribbon": {
    "hiddenItems": {
      "switcher:빠른 전환기 열기": false,
      "graph:그래프 뷰 열기": false,
      "canvas:새 캔버스 만들기": false,
      "daily-notes:오늘의 일일 노트 열기": false,
      "templates:템플릿 삽입": false,
      "command-palette:명령어 팔레트 열기": false,
      "bases:새 베이스 생성하기": false
    }
  },
  "active": "bases-leaf",
  "lastOpenFiles": [
    "10_Summary/종목_테이블.base"
  ]
}
```

- [ ] **Step 2: 내용 확인**

```powershell
Get-Content "C:\Users\김민철\Documents\TEST\.obsidian\workspace.json" | Select-String "종목_테이블"
```

Expected: `"10_Summary/종목_테이블.base"` 포함된 줄 2개 이상 출력

---

### Task 5: `stock_wiki/` 폴더 삭제

**Files:**
- Delete: `TEST/stock_wiki/` (전체 폴더)

- [ ] **Step 1: stock_wiki/ 삭제**

```powershell
Remove-Item -Recurse -Force "C:\Users\김민철\Documents\TEST\stock_wiki"
```

- [ ] **Step 2: 삭제 확인**

```powershell
Test-Path "C:\Users\김민철\Documents\TEST\stock_wiki"
```

Expected: `False`

---

### Task 6: git commit

**Files:** 위 Task 1-5에서 변경된 모든 파일

- [ ] **Step 1: 변경 사항 확인**

```powershell
git status
```

Expected: `.obsidian/` 신규 파일들, `10_Summary/종목_테이블.base` 신규, `stock_wiki/` 삭제됨

- [ ] **Step 2: 스테이징 및 커밋**

```powershell
git add .obsidian/ 10_Summary/종목_테이블.base
git rm -r stock_wiki/
git commit -m "feat: TEST/ 루트를 Obsidian 볼트로 설정, 종목 테이블 Bases 뷰 추가"
```

Expected: 커밋 성공

---

## 완료 후 확인

Obsidian에서 `TEST/` 폴더를 볼트로 열었을 때:
1. 좌측 파일 탐색기에 `10_Summary/`, `20_Stocks/`, `30_Daily_Log/`, `40_Trades/` 폴더가 보임
2. `docs/`, `.claude/` 폴더는 안 보임
3. 시작 화면에 종목 테이블 Bases 뷰가 열림
4. 테이블에 12개 종목 데이터가 표시됨

> **참고:** Obsidian Bases의 `.base` 파일 형식은 버전에 따라 다를 수 있음. 테이블이 정상 렌더링되지 않으면 Obsidian UI에서 직접 Bases 뷰를 재구성하면 됨.
