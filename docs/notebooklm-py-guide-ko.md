# notebooklm-py 전수조사 & 활용 가이드 (한국어)

> 작성일: 2026-09-21
> 작성: Claude Code 세션 대화 정리
>
> **원본 저장소:** <https://github.com/teng-lin/notebooklm-py>
> **포크 저장소:** <https://github.com/bmshin94/notebooklm-py>
> **PyPI:** <https://pypi.org/project/notebooklm-py/>
> **분석 대상 버전:** v0.8.2 (MIT License)

이 문서는 `notebooklm-py` 저장소를 전수조사한 결과와, 설치·사용법·정체성·수익화
아이디어까지 대화로 정리한 내용을 한 곳에 모은 것이다.

---

## 목차

1. [프로젝트 정체](#1-프로젝트-정체)
2. [규모와 폴더 구조](#2-규모와-폴더-구조)
3. [아키텍처](#3-아키텍처)
4. [기능 범위](#4-기능-범위)
5. [쉬운 설명: 왜 쓰는가](#5-쉬운-설명-왜-쓰는가)
6. [설치 및 사용법](#6-설치-및-사용법)
7. [플러그인 / 스킬 / MCP — 정체 정리](#7-플러그인--스킬--mcp--정체-정리)
8. [인증과 토큰](#8-인증과-토큰)
9. [왜 GitHub에서 유명한가](#9-왜-github에서-유명한가)
10. [로컬 에이전트 구축에서의 활용](#10-로컬-에이전트-구축에서의-활용)
11. [React / PHP로 만들 수 있는가](#11-react--php로-만들-수-있는가)
12. [수익화 아이디어](#12-수익화-아이디어)
13. [리스크 정리](#13-리스크-정리)

---

## 1. 프로젝트 정체

**Google NotebookLM(2026년 7월부터 "Gemini Notebook"으로 리브랜딩)을 코드로 조종하는
비공식 async Python 클라이언트.**

Google은 NotebookLM의 공식 API를 공개하지 않았다. 이 라이브러리는 NotebookLM 웹 UI가
내부적으로 사용하는 비공개 RPC 프로토콜 **`batchexecute`** 를 리버스 엔지니어링해서
직접 호출한다. 브라우저 자동화(Selenium/Playwright 클릭 방식)가 아니라 내부 API를
직접 때리는 방식이라 훨씬 빠르고 안정적이다.

| 항목 | 값 |
|---|---|
| 패키지명 | `notebooklm-py` |
| 버전 | 0.8.2 |
| 라이선스 | MIT |
| Python | 3.10 ~ 3.14 |
| 핵심 의존성 | `httpx`, `click`, `rich`, `filelock` |
| 콘솔 스크립트 | `notebooklm`, `notebooklm-mcp`, `notebooklm-server` |

---

## 2. 규모와 폴더 구조

### 규모

| 항목 | 수치 |
|---|---|
| Python 소스 파일 | 530개 |
| 소스 코드 라인 (`src/`) | 약 155,685줄 |
| 테스트 파일 | 872개 |
| 문서 | `docs/` 24개 + ADR 40개 |
| CI 워크플로우 | 15개 |
| 커버리지 게이트 | 90% 강제 |

### 폴더 구조

```
notebooklm-py/
├── src/notebooklm/
│   ├── client.py            NotebookLMClient — 진입점, 11개 타입드 네임스페이스
│   ├── raw.py               원시 RPC 탈출구 (저수준 직접 호출)
│   ├── auth.py / _auth/     구글 인증 (쿠키 · 마스터 토큰)
│   ├── _app/   (40개 모듈)  전송수단 중립 비즈니스 로직 — 핵심 두뇌
│   ├── _web/                Web 백엔드 (batchexecute / HTTP) — 기본값
│   ├── _android/            Android 백엔드 (gRPC + protobuf 스텁)
│   ├── _runtime/            이벤트루프 바인딩, 동시성 제어, 메트릭
│   ├── _browser/            Playwright 로그인 자동화
│   ├── _source/ _artifact/  소스 업로드 · 산출물 다운로드 파이프라인
│   ├── cli/    (50개 모듈)  `notebooklm` CLI (Click 기반)
│   ├── mcp/    (+tools/)    MCP 서버 (FastMCP) — 도구 41개
│   ├── server/ (+routes/)   REST 서버 (FastAPI, /v1)
│   └── rpc/_identifiers.py  난독화된 RPC 메서드 ID 61개 (최대 취약점)
│
├── SKILL.md                 Claude Code / Codex 용 에이전트 스킬 정의
├── AGENTS.md                Codex 용 저장소 가이드
├── CLAUDE.md                Claude Code 용 가이드
├── docs/                    설치 · CLI · Python API · 아키텍처 · MCP · 트러블슈팅
│   └── adr/                 설계 결정 기록 40건 (ADR-0000 ~ 0039)
├── tests/                   unit / integration(VCR) / e2e / qualification / guardrails
├── deploy/                  Docker + Tailscale Funnel (원격 MCP 배포)
├── desktop-extension/       MCPB 데스크톱 확장 (manifest.json)
├── examples/                bulk-import, chat, notes, research-to-podcast 등 7개
└── .github/workflows/       test, nightly, rpc-health, publish, codeql 등 15개
```

---

## 3. 아키텍처

6계층 소유권 구조.

```
[어댑터]   CLI(cli/) · MCP(mcp/) · REST(server/) · Python API
               │  (셋 다 동일한 두뇌를 공유)
[_app/]    전송 중립 비즈니스 로직 — 계획 수립 · ID 해석 · 재시도 · 에러 분류
               │
[client]   NotebookLMClient + 11개 타입드 네임스페이스
               │
[_runtime] 루프 바인딩 · 동시성 제어 · 타임아웃 예산 · 메트릭
               │
[백엔드]    _web (batchexecute)  또는  _android (gRPC)
               │
[전송]      Scotty 업로드 · Drive 스테이징 · 에셋 다운로드
```

| 어댑터 | 패키지 | 전송 | 콘솔 스크립트 | 설치 |
|---|---|---|---|---|
| CLI | `cli/` | 터미널 (Click) | `notebooklm` | 기본 |
| MCP | `mcp/` | Model Context Protocol (FastMCP) | `notebooklm-mcp` | `mcp` extra (실험적) |
| REST | `server/` | HTTP (FastAPI) | `notebooklm-server` | `server` extra (실험적) |

**핵심 설계 포인트:** CLI · MCP · REST가 전부 `_app/` 하나를 공유한다. 어느 경로로
쓰든 동작이 동일하고, 새 인터페이스를 붙이기도 쉽다. `_app/`은 transport 프레임워크를
import하지 않으며, 그 경계는 lint로 강제된다(`tests/_guardrails/test_app_boundary.py`).

### 11개 타입드 네임스페이스

`notebooks` · `sources` · `artifacts` · `chat` · `research` · `notes` ·
`mind_maps` · `settings` · `sharing` · `labels` · `collections`

백엔드 선택은 생성 시점에 한 번만 결정되며(`backend=` 인자 > `NOTEBOOKLM_BACKEND` >
기본값 `web`), 11개 네임스페이스 전체에 all-or-nothing으로 적용된다.

---

## 4. 기능 범위

| 카테고리 | 기능 |
|---|---|
| 노트북 | 생성 / 복사(소스 + 산출물 포함) / 목록 / 이름변경 / 삭제 |
| 소스 | URL, YouTube, 파일(PDF·txt·md·docx·EPUB·오디오·비디오·이미지), Google Drive, Google Play Books, 붙여넣기 텍스트, 새로고침, 전문 조회 |
| 채팅 | 인용 포함 질문, 대화 히스토리, 커스텀 페르소나, 추천 프롬프트, 백그라운드 비동기 질문 |
| 노트 | 생성 / 목록 / 이름변경 / 삭제 / 채팅 답변 저장 / 대화 전체 저장 |
| 라벨 | AI 자동 생성 또는 수동 토픽 라벨, 소스 멤버십 관리, 라벨 필터 |
| 리서치 | 웹 / Drive 리서치 에이전트 (fast · deep 모드), 결과 자동 임포트 |
| 공유 | 공개/비공개 링크, 사용자 권한(viewer/editor), 뷰 레벨 제어 |

### 콘텐츠 생성 (전체 산출물 타입)

| 타입 | 옵션 | 다운로드 포맷 |
|---|---|---|
| Audio Overview | 4개 포맷(deep-dive/brief/critique/debate), 3개 길이, 50개 이상 언어 | MP3 |
| Video Overview | 4개 포맷, 8개 비주얼 스타일 | MP4 |
| Slide Deck | detailed / presenter, 길이 조절, 개별 슬라이드 수정 | PDF, PPTX |
| Infographic | 3개 방향, 3개 상세도 | PNG |
| Quiz | 개수 · 난이도 설정 | JSON, Markdown, HTML |
| Flashcards | 개수 · 난이도 설정 | JSON, Markdown, HTML |
| Report | briefing-doc / study-guide / blog-post / custom | Markdown |
| Data Table | 자연어로 구조 지정 | CSV |
| Mind Map | note-backed JSON 또는 interactive studio map | JSON |

### 웹 UI로는 불가능한 기능 (이 라이브러리의 존재 이유)

1. **일괄 다운로드** — 특정 타입 산출물을 한 번에 전부
2. **퀴즈 / 플래시카드 내보내기** — JSON · Markdown · HTML (Anki 연동 가능)
3. **마인드맵 데이터 추출** — 계층 JSON으로 추출, 시각화 도구 연동
4. **데이터 테이블 CSV 내보내기**
5. **슬라이드 PPTX / PDF 다운로드**
6. **개별 슬라이드 자연어 수정**
7. **리포트 템플릿 커스터마이즈**
8. **채팅 히스토리 전체를 노트로 저장**
9. **소스 전문(indexed fulltext) 접근**
10. **프로그래매틱 공유 권한 관리**

---

## 5. 쉬운 설명: 왜 쓰는가

### NotebookLM이란

- ChatGPT / Claude → 인터넷 전체를 학습한 AI. 할루시네이션 가능성 있음.
- **NotebookLM** → 사용자가 넣어준 자료만 읽고, **출처를 인용해서** 답변함.

즉 NotebookLM은 "내 서류 가방만 보는 똑똑한 비서"이고, 그 비서가 서류를 읽고
팟캐스트 · 퀴즈 · 마인드맵까지 만들어준다.

### notebooklm-py는

> NotebookLM 비서에게 거는 **직통 전화선**.

웹 UI에서 30분 걸릴 일을 몇 줄로 끝낸다.

```bash
notebooklm create "9월 리서치"
notebooklm source add -n <ID> https://블로그1.com https://블로그2.com report.pdf
notebooklm ask -n <ID> "이 자료들의 공통 결론이 뭐야?"
notebooklm generate audio -n <ID> --wait && notebooklm download audio -n <ID>
```

### 4가지 사용 방식

| 방식 | 비유 | 대상 |
|---|---|---|
| Python API | 재료 사서 직접 요리 | 앱에 임베드하는 개발자 |
| CLI | 라면 끓이기 (한 줄 명령) | 터미널 사용자, 자동화 스크립트 |
| MCP 서버 | AI에게 주방 열쇠를 맡김 | Claude Code / ChatGPT 사용자 |
| REST 서버 | 배달 주문 창구 | 웹앱, 타 언어 연동 |

### 핵심 가치 4가지

1. **토큰 절감** — 무거운 문서 독해는 NotebookLM(Google)이 처리, LLM 에이전트는 최종
   마무리만. 이게 이 라이브러리의 킬러 패턴.
2. **에이전트 영구 기억** — "Master Brain" 노트북에 세션 결정사항을 노트로 적립하고,
   다음 세션 시작 시 `ask`로 회수. 벡터DB · 임베딩 파이프라인 불필요.
3. **인프라 0원 RAG** — 사내 문서 · RFC · 아키텍처 문서를 MCP로 노출하면 에이전트가
   추측 대신 인용 달린 사실로 답변.
4. **원소스 멀티유즈** — 자료 한 세트로 팟캐스트 · 영상 · 슬라이드 · 블로그 초안 ·
   퀴즈 · 플래시카드를 모두 생성.

---

## 6. 설치 및 사용법

### 설치 (권장)

```bash
# 0) uv 설치
curl -LsSf https://astral.sh/uv/install.sh | sh
#    Windows: winget install astral-sh.uv

# 1) CLI 설치 (격리 환경 + PATH 자동 등록)
uv tool install "notebooklm-py[browser]"
#    또는: pipx install "notebooklm-py[browser]"

# 2) 구글 로그인 (첫 실행 시 Chromium 약 170MB 자동 다운로드)
notebooklm login

# 3) 검증 — "status": "ok" 가 나와야 성공
notebooklm auth check --test --json
```

> `pip install`을 시스템 파이썬에 바로 쓰면 macOS(Homebrew) / Debian / Ubuntu에서
> PEP 668 `externally-managed-environment` 에러가 난다. venv 안에서 쓸 것.

### 라이브러리로 임베드

```bash
uv add notebooklm-py          # 또는 venv 안에서: pip install notebooklm-py
```

### 설치 옵션 (extras)

| extra | 내용 | 용도 |
|---|---|---|
| `browser` | Playwright | 로그인 자동화 (대부분 필요) |
| `cookies` | rookie-cookies | 이미 로그인된 브라우저에서 쿠키 추출 |
| `headless` | gpsoauth | 마스터 토큰 인증 (서버 / CI 필수) |
| `mcp` | fastmcp 3.4.2 (핀 고정) | MCP 서버 |
| `server` | FastAPI + uvicorn + python-multipart | REST 서버 |
| `android` | grpcio + protobuf + gpsoauth | Android 백엔드 |
| `markdown` | markdownify | 마크다운 변환 |
| `impersonate` | curl_cffi | TLS-JA3 임퍼소네이션 (실험적) |
| `all` | browser, dev, headless, markdown, mcp, server | 풀세트 (android/cookies/impersonate 제외) |

### 기본 워크플로우 — CLI

```bash
notebooklm create "AI 리서치" --json          # .notebook.id 확보
notebooklm source add -n <ID> https://arxiv.org/abs/xxxx paper.pdf
notebooklm source wait -n <ID>                # 인덱싱 완료 대기 (필수)
notebooklm ask -n <ID> "핵심 3가지만" --json
notebooklm generate audio -n <ID> --wait
notebooklm download audio -n <ID> -o ./podcast.mp3
```

### 기본 워크플로우 — Python API

```python
import asyncio
from notebooklm import NotebookLMClient

async def main() -> None:
    async with NotebookLMClient.from_storage() as client:
        nb = await client.notebooks.create("내 연구")
        src = await client.sources.add_url(nb.id, "https://example.com")
        await client.sources.wait_until_ready(nb.id, src.id)
        answer = await client.chat.ask(nb.id, "요약해줘")
        print(answer.text)

asyncio.run(main())
```

### 기본 워크플로우 — MCP (Claude Code 연동)

```bash
notebooklm mcp install claude-code   # MCP 설정 자동 작성
notebooklm skill install             # ~/.claude/skills/notebooklm 에 스킬 설치
# 클라이언트 재시작 필수 (대부분의 호스트는 시작 시에만 MCP 설정을 읽음)
```

### 에이전트 운용 시 주의 (SKILL.md 규약)

- 모든 discovery / mutation에 `--json`을 쓰고 반환된 **전체 UUID를 보관**한다.
- 노트북 스코프 명령에는 항상 `-n/--notebook <id>`를 넘긴다. `notebooklm use`에 의존하지 않는다.
- 동시 실행 시 에이전트마다 `NOTEBOOKLM_PROFILE=agent-<id>`로 프로필을 분리한다.
  **하나의 쓰기 가능한 `storage_state.json`을 여러 에이전트가 공유하면 안 된다.**
- 소스 추가 후 반드시 `source wait`로 `status == "ready"`를 확인한 뒤 채팅/생성을 한다.

---

## 7. 플러그인 / 스킬 / MCP — 정체 정리

**정답: 전부 다다.** 하나의 본체에 여러 껍데기가 붙어 있는 구조.

| 정체 | 위치 | 설명 |
|---|---|---|
| Python 라이브러리 | PyPI `notebooklm-py` | **본체.** 나머지는 전부 이것을 감싼 것 |
| CLI 툴 | `notebooklm` | 터미널 어댑터 |
| MCP 서버 | `notebooklm-mcp` (`mcp` extra) | 도구 41개 노출. Claude / ChatGPT가 직접 호출 |
| Agent Skill | 루트 `SKILL.md` | Claude Code / Codex / `.agents` 스킬 디렉터리 |
| REST 서버 | `notebooklm-server` (`server` extra) | FastAPI `/v1` |
| 데스크톱 확장 | `desktop-extension/manifest.json` | MCPB 번들 |
| Docker 배포 | `deploy/` | 원격 MCP 커넥터 (Cloudflare / Tailscale 터널) |

**"플러그인"은 아니다.** Claude Code 플러그인 규격(`.claude-plugin/`)은 없고,
**스킬 + MCP** 두 갈래로 에이전트에 붙는다.

### MCP 도구 41개

```
notebook_list / notebook_create / notebook_describe / notebook_rename / notebook_delete
source_list / source_read / source_add / source_rename / source_delete / source_wait
source_add_drive_file / source_add_play_book / source_list_play_books / await_upload
chat_ask / chat_start / chat_status / chat_cancel / chat_configure / suggest_prompts
note_save
studio_list / studio_generate / studio_status / studio_download
studio_rename / studio_retry / studio_delete
research_start / research_status / research_import / research_cancel
share_status / share_set_access / share_set_user / share_remove_user
server_info
```

읽기 전용 도구에는 `READ_ONLY`, 삭제성 도구(`*_delete` 3종 + `share_remove_user`)에는
`DESTRUCTIVE` 어노테이션이 붙고 `confirm` 인자를 요구한다. MCP 어노테이션을 존중하는
호스트는 읽기는 자동 승인, 삭제는 게이팅할 수 있다.

---

## 8. 인증과 토큰

**Google 공식 API 키는 존재하지 않는다.** 대신 3가지 인증 방식이 있다.

| 방식 | 명령 | 저장 파일 | 특징 |
|---|---|---|---|
| 브라우저 로그인 | `notebooklm login` | `storage_state.json` | 기본값. Playwright가 창을 띄워 구글 로그인 |
| 기존 쿠키 임포트 | `login --browser-cookies chrome` | `storage_state.json` | Playwright 불필요. 로그인된 브라우저에서 추출 |
| **마스터 토큰** | `login --master-token --account you@example.com` | `master_token.json` | 쿠키를 필요할 때마다 자동 재발급. **서버 / CI / 무인 운영의 정답** |

저장 위치: `~/.notebooklm/profiles/<프로필>/`
(`NOTEBOOKLM_HOME`으로 베이스 디렉터리, `NOTEBOOKLM_PROFILE`로 프로필 선택)

### 보안 경고

> **마스터 토큰은 비밀번호를 변경해도 살아남는, 구글 계정 전체 권한 자격증명이다.**
>
> - 절대 출력 / 로그 / 커밋 금지
> - 파일 권한 `0600` 필수
> - **전용 서브 계정** 사용 강력 권장
> - 유출 시 즉시 구글 계정에서 명시적으로 revoke

### 별도로 사용자가 정하는 토큰

| 토큰 | 용도 |
|---|---|
| `NOTEBOOKLM_SERVER_TOKEN` | REST 서버 접근 인증 (필수, 상수시간 비교) |
| MCP 원격 Bearer / OAuth | 원격 MCP 커넥터 (claude.ai / ChatGPT 연결) |
| `NOTEBOOKLM_AUTH_JSON` | CI용 인라인 쿠키 JSON (단기 대체수단, 자동 복구 경로 우회) |
| `NOTEBOOKLM_MASTER_TOKEN_JSON` | CI 시크릿 전달 관례. 패키지가 직접 읽지 않고, 프로필의 `master_token.json`(0600)으로 써야 함 |

---

## 9. 왜 GitHub에서 유명한가

1. **공식 API 부재 — 사실상 유일한 대안.** NotebookLM 자동화 수요는 큰데 공급이
   이것 하나뿐이라 독점적 생태 지위를 가진다.
2. **AI 에이전트 붐과의 정확한 교차.** MCP 서버와 Agent Skill을 동시에 제공하면서
   Claude Code / MCP 폭발 시기에 정확히 착지했다. Trendshift에도 등재되었다
   (repository 19116).
3. **토큰 절감 = 비용 절감 서사.** "문서 30개를 LLM에 직접 넣으면 요금 폭탄,
   NotebookLM에 넣으면 사실상 무료"라는, 개발자가 가장 좋아하는 종류의 이야기.
4. **웹 UI가 못 하는 것을 한다.** 마인드맵 JSON, 퀴즈→Anki, PPTX 다운로드, 일괄
   내보내기 등 명확한 차별점.
5. **압도적 엔지니어링 완성도.** 테스트 872개 파일, 커버리지 90% 게이트, ADR 40건,
   문서 24개 + GitHub Pages 다이어그램, CI 15개(RPC health 감시 포함),
   Python 3.10~3.14 지원. 비공식 리버스 엔지니어링 프로젝트가 이 정도 규율을 갖춘 건
   매우 드물다.
6. **Obsidian / 지식그래프 커뮤니티 흡수.** 볼트 루트에서 CLI를 돌리면 산출물이 그대로
   파일로 떨어지고, 인용 마커를 위키링크로 변환하는 커뮤니티 스킬까지 파생되었다.

---

## 10. 로컬 에이전트 구축에서의 활용

**결론: 매우 유용하다. 이 라이브러리 존재 이유의 절반이 여기에 있다.**

| 용도 | 설명 |
|---|---|
| 제로 인프라 RAG | 임베딩 모델 · 벡터DB · 청킹 · 리랭커 · GPU 없이 인용 포함 검색 |
| 에이전트 장기 기억 | 세션 종료 시 `note create`로 적립, 세션 시작 시 `ask`로 회수 |
| 컨텍스트 윈도우 우회 | 에이전트가 담을 수 없는 방대한 문서를 "질의용 신탁"으로 운용 |
| 스킬 자가검증 | NotebookLM이 소스로부터 퀴즈(eval set)를 생성 → 에이전트 스킬 채점 → 반복 개선 |
| 모바일 운용 | `deploy/` Docker + Tailscale Funnel → claude.ai 커스텀 커넥터 → 폰에서 전체 툴셋 사용 |

### 주의사항

- **네트워크 의존** — 완전 오프라인 로컬 에이전트에는 부적합.
- **레이트 리밋** — 계정 등급별 쿼터 존재 (`docs/quota-limits.md`).
- **동시성 규칙** — 에이전트마다 `NOTEBOOKLM_PROFILE` 분리 필수.
- **루프 어피니티** — 클라이언트 1개 = 이벤트루프 1개 = 스레드 1개. 루프 간 재사용 금지.

---

## 11. React / PHP로 만들 수 있는가

### 권장: 포팅하지 말고 감싼다

```
[React 프론트엔드] --HTTP--> [notebooklm-server (FastAPI /v1)] --> NotebookLM
   Vite / Next                NOTEBOOKLM_SERVER_TOKEN 인증

[PHP (Laravel)]  --Guzzle--> [notebooklm-server] --> NotebookLM
```

`server/routes/`에 노트북 · 소스 · 채팅 · 산출물 · 리서치 · 공유 라우트가 이미 전부
구현되어 있으므로 React/PHP는 HTTP 호출만 하면 된다.

### 직접 재구현(순수 JS / PHP 포팅)을 권하지 않는 이유

| 이유 | 설명 |
|---|---|
| CORS | 브라우저 React에서 구글 내부 API 직접 호출은 불가능. 서버가 반드시 필요 |
| 쿠키 인증 | 구글 세션 쿠키를 브라우저 JS가 다룰 수 없음 |
| RPC ID 추적 | 난독화 ID 61개가 수시로 변경됨. 혼자서 계속 따라잡아야 함 |
| 위치 민감 중첩 파라미터 | 소스 ID 중첩 깊이가 API마다 다름: `[id]` / `[[id]]` / `[[[id]]]` / `[[[[id]]]]` |
| 규모 | 155,000줄. 재시도 · 멱등성 · 업로드 파이프라인 · 에러 분류를 다시 만들면 수개월 |

### 실무 팁

- 프로덕션은 `deploy/Dockerfile`로 REST 서버를 컨테이너화하고 React는 별도 배포.
- REST `/v1`은 **experimental**이라 마이너 버전에서 변경될 수 있다. 버전 핀 고정 권장.
- 생성 작업은 비동기(HTTP 202 + `task_id`)이므로 프론트에 폴링 UI가 필요하다.

---

## 12. 수익화 아이디어

> 전제: 이것은 비공식 라이브러리이며 Google ToS상 회색지대다. 상업화 시
> ① RPC 변경으로 인한 서비스 중단, ② 계정 차단, ③ ToS 이슈 가능성이 있다.
> 아래는 **리스크가 낮은 순**으로 정렬했다.

### 티어 1 — 리스크 낮음, 즉시 가능

#### ① 지식 · 교육 콘텐츠 판매 (최우선 추천)

- 유튜브 / 블로그 (관련 콘텐츠가 이미 높은 조회수를 기록 중)
- 유료 강의: "AI 에이전트 토큰 90% 줄이기"
- 유료 뉴스레터 / 노션 템플릿 팩

계정 리스크 0, 초기 비용 0, 구글이 API를 바꿔도 콘텐츠 가치는 유지된다.

#### ② 에이전트 "스킬 팩" 판매 (가장 유망)

NotebookLM으로 특정 도메인 문서를 증류해 만든 `SKILL.md` 패키지를 판매한다.

```
"React 19 + Next 15 마스터 스킬팩"   $29
"AWS 인프라 베스트프랙티스 스킬팩"   $39
"한국 세무 / 노무 실무 스킬팩"       $49
```

메커니즘:
1. 공식 문서 수백 페이지를 노트북에 투입
2. NotebookLM이 증류 → `SKILL.md` 생성
3. NotebookLM이 퀴즈를 생성해 **자가 검증**
4. 한 번 만들면 런타임 토큰 0, 네트워크 호출 0 — 구매자는 파일만 받으면 끝

구매자 측에 라이브러리조차 필요 없으므로 **리스크가 완전히 격리**된다.

#### ③ 프리랜스 / 컨설팅 자동화

기업 내부 문서를 사내 Q&A 봇으로 구축해주는 대행. 온보딩 자료를 교육용 팟캐스트 +
퀴즈 파이프라인으로 자동 변환. 납품물을 **고객 계정 + 고객 인프라**로 구성하면
리스크가 고객 측으로 이전된다.

### 티어 2 — 중간 리스크, BYOA 모델

> **BYOA (Bring Your Own Account)** — 사용자가 자기 구글 계정으로 로그인하게 만드는
> 것이 핵심. 내 계정을 쓰지 않으므로 차단 리스크가 분산된다.

#### ④ 데스크톱 앱 (로컬 실행형) — 리스크 대비 효율 최고

Electron / Tauri + React GUI, 내부에 `notebooklm-server` 번들.

```
기능: 드래그앤드롭 일괄 업로드 · 산출물 갤러리 · 일괄 다운로드
      Obsidian 볼트 동기화 · 예약 팟캐스트 생성 · 다계정 프로필 전환
가격: 라이선스 $49 평생 / $9 월
```

전부 사용자 PC에서 실행되므로 서버도 없고 타인의 자격증명도 보관하지 않는다.
법적 · 보안 리스크가 최소다.

#### ⑤ Obsidian / Notion 플러그인 (프리미엄)

- 무료: 기본 `ask`
- 프로 $5/월: 일괄 임포트, 마인드맵 → 캔버스 변환, 인용 → 위키링크 자동 해석,
  오디오 다이제스트

#### ⑥ 니치 SaaS (수직 특화)

| 타깃 | 상품 | 가격 |
|---|---|---|
| 수험생 / 학생 | 강의자료 → 팟캐스트 + 퀴즈 + Anki 덱 | $9/월 |
| 로펌 / 세무 | 판례 · 법령 근거 인용 Q&A | $99/월 |
| 의료 | 논문 → 요약 브리핑 자동 생성 | $79/월 |
| 마케팅 | 자료 → 블로그 + 영상 + 슬라이드 원소스 멀티유즈 | $29/월 |
| 팟캐스터 | 뉴스 크롤링 → 매일 아침 오디오 브리핑 자동 발행 | $19/월 |

특히 오디오 브리핑 자동화는 `auth refresh --quiet`(cron) + `generate audio` 조합으로
완전 무인 운영이 가능하다.

#### ⑦ 원격 MCP 커넥터 호스팅 (B2B)

`deploy/`의 Docker + Tailscale Funnel 구성을 팀용 관리형 서비스로 판매.
"팀 지식베이스를 MCP로 — 모든 팀원의 Claude/ChatGPT가 사내 문서를 인용해 답변"
($199/월, 팀 10명). 단, 고객 자격증명을 보관하게 되므로 보안 설계가 필수다.

### 티어 3 — 고위험 (신중히)

- **⑧ 완전 관리형 SaaS (운영자 계정 풀 사용)** — 계정 밴 시 전 고객 서비스 중단,
  쿼터 한계, ToS 정면충돌. 비추천.
- **⑨ API 재판매** — NotebookLM을 감싸 "NotebookLM API"로 되파는 형태. 가장 위험.

### 권장 로드맵

```
1개월차   콘텐츠 (유튜브 / 블로그)      — 무료, 리스크 0, 시장 반응 테스트
3개월차   스킬팩 판매 (Gumroad)         — 첫 수익, 여전히 리스크 0
6개월차   데스크톱 앱 / Obsidian 플러그인 — MRR 시작, BYOA로 리스크 분산
12개월차  니치 SaaS / B2B 컨설팅        — 본격 스케일
```

### 성공 조건 3가지

1. **"NotebookLM 래퍼"로 팔지 않는다.** "리서치 자동화 도구"로 포지셔닝해야 구글이
   무엇을 바꿔도 가치가 유지된다.
2. **BYOA를 기본으로 한다.** 사용자 자기 계정으로 로그인하게 만든다.
3. **폴백을 준비한다.** 라이브러리가 깨져도 서비스가 죽지 않도록, 핵심 가치가
   워크플로우 쪽에 있어야 한다.

---

## 13. 리스크 정리

| 리스크 | 내용 | 완화 |
|---|---|---|
| **RPC ID 변경** | `rpc/_identifiers.py`의 난독화 ID 61개를 구글이 바꾸면 라이브러리 전체가 깨짐 | 업스트림 `rpc-health.yml` CI가 상시 감시. 버전 업그레이드 추적 |
| **위치 민감 중첩 파라미터** | 소스 ID 중첩 깊이가 API마다 다름 | 기존 shape를 그대로 미러링 |
| **CSRF 토큰 만료** | 세션 만료 | `client.refresh_auth()` 또는 `notebooklm login` 재실행 |
| **레이트 리밋** | 계정 등급별 쿼터 | 대량 작업 사이에 지연 삽입 |
| **동시성** | 클라이언트 1개가 `open()` 시점 이벤트루프에 바인딩됨 | 스레드당 1개, 루프/테넌트 간 재사용 금지 |
| **마스터 토큰 유출** | 계정 전체 권한 탈취 | 전용 계정, `0600`, 시크릿 스토어, 즉시 revoke |
| **ToS / 계정 차단** | 비공식 API 사용 | BYOA 모델, 상업화 시 법률 검토 |

---

## 참고 링크

- 원본 저장소: <https://github.com/teng-lin/notebooklm-py>
- 포크 저장소: <https://github.com/bmshin94/notebooklm-py>
- PyPI: <https://pypi.org/project/notebooklm-py/>
- 설치 가이드: [docs/installation.md](installation.md)
- CLI 레퍼런스: [docs/cli-reference.md](cli-reference.md)
- Python API: [docs/python-api.md](python-api.md)
- 아키텍처: [docs/architecture.md](architecture.md)
- MCP 가이드: [docs/mcp-guide.md](mcp-guide.md)
- 트러블슈팅: [docs/troubleshooting.md](troubleshooting.md)
- 쿼터 제한: [docs/quota-limits.md](quota-limits.md)
- 보안: [docs/security.md](security.md)
- ADR 목록: [docs/adr/](adr/)
