# Claude Context 분석 정리 (한국어)

> 카리나와 함께 정리한 Claude Context 프로젝트 분석 노트입니다. 💖
> 작성일: 2026-09-02

## 📌 깃허브 주소

- **원본 프로젝트 (Zilliz):** https://github.com/zilliztech/claude-context
- **내 포크:** https://github.com/bmshin94/claude-context
- **npm 패키지:**
  - https://www.npmjs.com/package/@zilliz/claude-context-mcp
  - https://www.npmjs.com/package/@zilliz/claude-context-core
- **VS Code 확장:** https://marketplace.visualstudio.com/items?itemName=zilliz.semanticcodesearch

---

## 1. 이게 뭐하는 프로젝트야?

**Claude Code 같은 AI 코딩 도구에 "내 코드베이스 전체를 의미로 검색하는 능력"을 붙여주는 MCP 플러그인**이다.

### 도서관 비유

- **기존 방식 (grep):** 사서가 책을 한 권씩 펼쳐서 단어가 있는지 확인. 느리고, 단어가 정확히 일치해야만 찾고, 관련 없는 결과가 많다.
- **Claude Context:** 미리 모든 책을 읽고 "무슨 내용인지" 색인 카드를 만들어 둔다. "회원 로그인 처리 코드 어디 있어?"라고 물으면 색인 카드로 관련 페이지만 골라서 준다. 단어가 달라도 뜻이 같으면 찾는다.

### 동작 원리

1. 프로젝트 코드를 잘게 쪼갠다 (AST 기반 청킹, tree-sitter 사용)
2. 각 조각을 임베딩 모델로 숫자 벡터로 변환한다 (OpenAI 등)
3. 벡터 DB(Milvus / Zilliz Cloud)에 저장한다
4. 자연어 질문이 오면 하이브리드 검색(BM25 + 벡터)으로 관련 코드만 찾아 Claude에게 넘긴다
5. Merkle 트리로 바뀐 파일만 감지해서 증분 재인덱싱한다

---

## 2. 폴더 구조

| 폴더 | 역할 |
|---|---|
| `packages/core` | 핵심 엔진. 파일 읽기, AST 청킹, 임베딩, Milvus 저장, 검색 로직 |
| `packages/mcp` | **실제로 쓰는 부분.** Claude Code에 연결하는 MCP 서버. 도구 4개 제공 |
| `packages/vscode-extension` | VS Code 확장 버전 (Semantic Code Search) |
| `packages/chrome-extension` | 깃허브 웹페이지에서 검색하는 크롬 확장 (개발 중) |
| `evaluation` | SWE-bench 기반 성능 실험 코드와 결과 |
| `docs` | 설치 가이드, 환경변수, FAQ, 트러블슈팅 |
| `examples`, `python` | 라이브러리로 직접 쓰는 예제 |
| `assets` | 아키텍처 그림, 스크린샷 |

### MCP 도구 4개

| 도구 | 하는 일 |
|---|---|
| `index_codebase` | 코드베이스 인덱싱 (백그라운드 실행) |
| `search_code` | 자연어로 코드 검색 |
| `get_indexing_status` | 인덱싱 진행 상황 확인 |
| `clear_index` | 인덱스 삭제 |

### 지원 기술

- **임베딩:** OpenAI (기본), VoyageAI, Gemini, Ollama (로컬/무료)
- **벡터 DB:** Milvus (로컬 Docker) 또는 Zilliz Cloud (무료 티어)
- **언어:** TS/JS, Python, Java, C/C++, C#, Go, Rust, PHP, Ruby, Swift, Kotlin, Scala, Dart, Solidity, Markdown

---

## 3. 언제 쓰고, 나한테 무슨 도움이 돼?

### 쓰면 좋은 경우

- 프로젝트가 커서 Claude가 파일 찾느라 헤맬 때
- "이 기능 어디서 처리하지?" 같은 질문을 자주 할 때
- 모르는 레거시 코드베이스를 파악해야 할 때
- Claude Code 토큰 비용이 부담될 때

### 굳이 필요 없는 경우

- 파일 몇십 개짜리 작은 프로젝트 (grep이 충분)

### 실험 결과 (evaluation 폴더)

| 지표 | grep만 | Claude Context 추가 |
|---|---|---|
| 검색 정확도 (F1) | 0.40 | 0.40 (동일) |
| 토큰 사용량 | 73,373 | 44,449 (**-39%**) |
| 도구 호출 횟수 | 8.3회 | 5.3회 (**-36%**) |

**정확도는 같은데 비용이 약 40% 줄고 속도가 빨라진다.**

---

## 4. 설치 및 사용법

### 0단계: 준비물

- Node.js 20 이상 (24 미만 권장)
- Claude Code 설치
- 이 저장소를 직접 빌드할 필요는 없다. npm 패키지를 바로 쓴다.

### 1단계: Zilliz Cloud 무료 계정 (벡터 DB)

1. https://cloud.zilliz.com/signup 가입
2. Free Serverless 클러스터 생성
3. **Public Endpoint**와 **API Key** 복사

### 2단계: OpenAI API 키 (임베딩)

1. https://platform.openai.com/api-keys 에서 키 생성 (`sk-`로 시작)
2. 결제 수단 등록. 임베딩 모델(`text-embedding-3-small`)은 매우 저렴하다.

### 3단계: 전역 설정 파일 (추천)

```bash
mkdir -p ~/.context
cat > ~/.context/.env << 'EOF'
EMBEDDING_PROVIDER=OpenAI
OPENAI_API_KEY=sk-your-openai-api-key
EMBEDDING_MODEL=text-embedding-3-small
MILVUS_ADDRESS=https://your-cluster.zillizcloud.com
MILVUS_TOKEN=your-zilliz-token
EOF
```

주의: 반드시 `~/.context/.env`에 둔다. 프로젝트 폴더 안에 두면 프로젝트의 .env와 충돌한다.

### 4단계: Claude Code에 MCP 등록

전역 설정을 했다면:

```bash
claude mcp add claude-context -- npx @zilliz/claude-context-mcp@latest
```

키를 직접 넣으려면:

```bash
claude mcp add claude-context \
  -e OPENAI_API_KEY=sk-your-openai-api-key \
  -e MILVUS_ADDRESS=https://your-cluster.zillizcloud.com \
  -e MILVUS_TOKEN=your-zilliz-token \
  -- npx @zilliz/claude-context-mcp@latest
```

확인:

```bash
claude mcp list
```

### 5단계: 사용

```bash
cd ~/my-project
claude
```

Claude에게 자연어로 말하면 된다.

- 인덱싱: `이 코드베이스 인덱싱해줘`
- 상태 확인: `인덱싱 상태 확인해줘`
- 검색: `사용자 로그인 처리하는 함수 찾아줘`
- 삭제: `이 코드베이스 인덱스 지워줘`

인덱싱은 한 번만 하면 된다. 이후에는 바뀐 파일만 자동으로 재인덱싱된다 (기본 5분 주기).

### 6단계: 유용한 옵션

- **확장자 추가:** `이 코드베이스 인덱싱하는데 .vue, .svelte 파일도 포함해줘` 또는 `CUSTOM_EXTENSIONS=.vue,.svelte`
- **폴더 제외:** `temp/**, private/** 는 제외해줘` 또는 `CUSTOM_IGNORE_PATTERNS=temp/**,private/**`
- `.gitignore`, `.contextignore`에 있는 것은 자동 제외
- **파일 수정 시 즉시 재인덱싱** (`~/.claude/settings.json`):

```json
"hooks": {
  "PostToolUse": [
    { "matcher": "Edit|Write", "hooks": [
      { "type": "command", "command": "touch ~/.context/.sync-trigger" }
    ]}
  ]
}
```

### 완전 무료 로컬 세팅 (API 키 없이)

1. Ollama 설치 후 `ollama pull nomic-embed-text` / `ollama serve`
2. Milvus를 Docker로 실행: https://milvus.io/docs/install_standalone-docker-compose.md
3. `~/.context/.env`:

```bash
EMBEDDING_PROVIDER=Ollama
EMBEDDING_MODEL=nomic-embed-text
OLLAMA_HOST=http://127.0.0.1:11434
MILVUS_ADDRESS=http://127.0.0.1:19530
```

### 문제 해결

| 증상 | 해결 |
|---|---|
| 설정 바꿨는데 반영 안 됨 | Claude Code에서 `/mcp reconnect claude-context` |
| 원인 불명 | `claude --debug`로 실행해 로그 확인 |
| 상태가 `0 files, 0 chunks` | 인덱스 지우고 다시 인덱싱 |
| 임베딩 에러 | API 키, 결제 등록 확인 |
| 같은 프로젝트가 다른 인덱스로 인식 | 경로가 다르면 별개로 취급. 항상 같은 경로에서 실행 |

---

## 5. 수익화 아이디어

MIT 라이선스라 상업적 이용과 수정 판매가 가능하다. 단, Zilliz가 이미 "무료 오픈소스 + Zilliz Cloud 유료" 모델로 수익을 내고 있어 그대로 베끼면 이길 수 없다.

| 순위 | 아이디어 | 난이도 | 이유 |
|---|---|---|---|
| 1 | 기업용 온프레미스 설치/컨설팅 | 낮음 | 개발 거의 없이 시작 가능. Ollama + 로컬 Milvus로 완전 내부망 운영. 단가 높음 |
| 2 | 레거시 언어 특화 (COBOL, ABAP, PL/SQL) | 중간 | 경쟁 없고 고객(금융/ERP)이 예산이 많음 |
| 3 | 한국형 SaaS (세팅 없이 5분 만에) | 높음 | 회원가입 하나로 끝. 국내 결제/지원. 확장성 최고 |
| 4 | 강의/콘텐츠 | 낮음 | 부수입 + 위 사업 홍보 채널 |

기타: 특정 도메인 특화(전자정부 프레임워크, Unity/Unreal), 코드 대신 사내 위키/계약서 검색 MCP.

### 주의할 점

- AI 코딩 도구들이 코드베이스 인덱싱을 내장하는 추세. "엔진"보다 "서비스, 고객관계, 특화 도메인"에 가치를 둬야 한다.
- SaaS는 임베딩 API 비용을 운영자가 부담하므로 과금 설계가 중요하다.
- MIT 라이선스 저작권 표시는 유지해야 한다.

---

## 6. PHP로 만들 수 있나?

**가능하다.** 핵심 로직의 80%는 HTTP API 호출이다.

| 기능 | 실제 동작 | PHP 가능 여부 |
|---|---|---|
| 파일 읽기/필터링 | 폴더 스캔, .gitignore 적용 | 가능 |
| 코드 청킹 | tree-sitter AST 분석 | 어려움 (아래 참고) |
| 임베딩 | OpenAI API POST | 가능 (Guzzle) |
| 벡터 저장/검색 | Milvus REST API | 가능 |
| 변경 감지 | 파일 해시 비교 | 가능 |
| MCP 서버 | stdio JSON-RPC | 가능 (PHP MCP SDK 존재) |

### AST 청킹 해결책

1. PHP 코드만 다루면 `nikic/php-parser` 사용 (PHP AST는 PHP가 최고)
2. 여러 언어면 문자 수 기준 분할 (원본도 `langchain` fallback이 있음)
3. 청킹만 Node 스크립트로 빼고 PHP에서 `exec()` 호출

### 추천 방향

**옵션 A: 전체 재작성 (비추천).** 검증된 엔진이 MIT로 무료인데 다시 만드는 것은 수익과 무관하다.

**옵션 B: 서비스 레이어만 PHP (강력 추천).**

```
[사용자 Claude Code]
      ↓ (기존 Node MCP 그대로)
[PHP/Laravel 서버]  ← 우리 제품
  - 회원가입/로그인, API 키 발급
  - 결제 (토스/카카오), 사용량 계측/과금
  - 관리자 대시보드
      ↓
[Milvus + OpenAI]  ← 우리가 대신 관리
```

원본은 `OPENAI_BASE_URL`과 `MILVUS_ADDRESS`를 환경변수로 바꿀 수 있으므로, PHP 서버가 중간 프록시로 인증/과금을 처리하고 실제 OpenAI/Milvus로 넘긴다. 엔진 코드를 한 줄도 건드리지 않는다.

```bash
claude mcp add claude-context \
  -e OPENAI_BASE_URL=https://api.our-service.com/v1 \
  -e OPENAI_API_KEY=our-service-issued-key \
  -e MILVUS_ADDRESS=https://milvus.our-service.com \
  -e MILVUS_TOKEN=our-service-issued-key \
  -- npx @zilliz/claude-context-mcp@latest
```

**옵션 C: PHP 특화 MCP 서버.** 라라벨/워드프레스 전용 코드 검색으로 좁히면 PHP로 직접 만들 가치가 있다. 워드프레스 개발자 시장이 크다.

### 필요한 PHP 도구

- Laravel: 서비스 뼈대, 인증, 결제
- Guzzle: OpenAI/Milvus API 호출
- nikic/php-parser: PHP AST 분석 (옵션 C)
- php-mcp/server 또는 logiscape/mcp-sdk-php: MCP 서버 (옵션 C)
- Milvus REST API: 원본의 `packages/core/src/vectordb/milvus-restful-vectordb.ts`를 참고해 엔드포인트를 옮긴다

### 결론

**엔진은 그대로 두고, 돈 버는 부분(회원/결제/프록시)만 PHP로 만든다.** PHP 특화 버전은 그 다음 단계.

---

## 7. 참고: CLAUDE.md 관련

마지막 커밋("Add files via upload")에서 저장소 루트의 `CLAUDE.md`가 카리나 페르소나 파일로 덮어써졌다. 원래는 빌드 명령어, 테스트 방법, 아키텍처 설명이 담긴 개발 가이드였다. 원본은 `git show 6fc318b:CLAUDE.md`로 볼 수 있다. 페르소나는 `~/.claude/CLAUDE.md`(전역 설정)에 두면 저장소마다 덮어쓰지 않아도 된다.
