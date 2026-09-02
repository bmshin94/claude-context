# Claude Context 분석 정리 & 수익화 아이디어

> 이 문서는 `claude-context` 저장소를 직접 뜯어보며 정리한 내용이다.
> 무엇을 하는 도구인지, 어떻게 설치·사용하는지, 그리고 이걸로 무엇을 만들 수 있는지를 담았다.

## 📌 저장소 주소

| 구분 | 주소 |
|---|---|
| **내 저장소 (fork)** | https://github.com/bmshin94/claude-context |
| **원본 저장소 (upstream)** | https://github.com/zilliztech/claude-context |
| npm - core | https://www.npmjs.com/package/@zilliz/claude-context-core |
| npm - mcp | https://www.npmjs.com/package/@zilliz/claude-context-mcp |
| VSCode 확장 | https://marketplace.visualstudio.com/items?itemName=zilliz.semanticcodesearch |
| 라이선스 | MIT (Copyright (c) 2025 Zilliz) — 포크·수정·상용화 자유, 저작권 고지만 유지 |

---

## 1. 이게 뭐하는 물건인가

한 줄 요약: **코드베이스 전체를 AI가 이해하는 검색엔진으로 만들어주는 MCP 플러그인.**

비유하자면 —

- **설치 전**: "사랑 이야기 책 찾아줘" → 알바생이 제목에 '사랑'이 들어간 책을 하나하나 뒤짐 (= `grep`, 단순 문자 매칭)
- **설치 후**: 사서가 미리 모든 책을 읽고 분류해둠 → 제목에 '사랑'이 없어도 로맨스 책을 바로 꺼내줌 (= 시맨틱 검색)

즉 함수 이름을 몰라도 **"장바구니 담는 기능 찾아줘"** 같은 자연어로 코드를 찾을 수 있게 해준다.

### 실측 효과 (`evaluation/README.md`)

SWE-bench Verified에서 30개 문제를 뽑아 3회씩 반복 측정한 결과:

| 항목 | grep만 사용 | Claude Context 사용 | 변화 |
|---|---|---|---|
| 평균 토큰 사용량 | 73,373 | 44,449 | **-39.4%** |
| 평균 툴 호출 횟수 | 8.3 | 5.3 | **-36.3%** |
| 검색 품질 (F1) | 0.40 | 0.40 | 동일 |

**품질 손해 없이 비용 40% 절감.** 코드베이스가 클수록 이득이 커진다.

---

## 2. 폴더 구조

pnpm 모노레포. 패키지 4개 + 부속 폴더.

| 경로 | 역할 |
|---|---|
| `packages/core/` | **심장부.** 코드 청킹 + 임베딩 + 벡터DB 저장/검색 엔진 |
| `packages/mcp/` | **실제로 쓰는 부분.** Claude Code에 붙는 MCP 서버 |
| `packages/vscode-extension/` | VSCode 확장판 ("Semantic Code Search") |
| `packages/chrome-extension/` | 크롬 확장 (private, 실험적) |
| `evaluation/` | 성능 실험 코드와 결과 (Python) |
| `docs/` | 설치·환경변수·트러블슈팅 문서 |
| `examples/basic-usage/` | core 패키지 직접 사용 예제 |

### `packages/core/src` 주요 파일

- `splitter/ast-splitter.ts` — tree-sitter로 AST 파싱해 **함수/클래스 단위**로 청킹 (기본 청크 2500자, 실패 시 자동 폴백)
- `splitter/langchain-splitter.ts` — 문자 기반 폴백 스플리터 (131줄)
- `embedding/` — OpenAI / VoyageAI / Gemini / **Ollama(로컬)** 4종 지원
- `vectordb/milvus-vectordb.ts` — gRPC 방식
- `vectordb/milvus-restful-vectordb.ts` — **HTTP만 쓰는 REST 방식** (제한된 환경용, 나중에 중요해짐)
- `sync/merkle.ts` + `sync/synchronizer.ts` — 머클 트리로 파일 해시 비교 → **바뀐 파일만 재색인** (스냅샷: `~/.context/merkle/`)

### 동작 파이프라인

```
코드 → AST 단위 청킹 → 임베딩(벡터 변환) → Milvus 저장
                                              ↓
"인증 로직 어디있어?" → 하이브리드 검색(BM25 + 밀집벡터, RRF 재랭킹) → 관련 코드만 반환
```

### 지원 언어

- **AST 파싱**: TypeScript, JavaScript, Python, Java, C++, C#, Go, Rust, Scala
- **문자 기반**: PHP, Ruby, Swift, Kotlin, Dart, Solidity, Markdown 등
- ⚠️ **PHP는 AST 지원 목록에 없다** (나중에 기회 포인트로 이어짐)

---

## 3. 설치 방법

### 준비물

- Node.js 20 이상
- Claude Code
- "임베딩(사서)" + "벡터DB(창고)" 각각 하나씩

| | 🅰️ 클라우드 방식 | 🅱️ 완전 로컬 방식 |
|---|---|---|
| 난이도 | 쉬움 | 번거로움 |
| 비용 | 소액 | 무료 |
| 추천 대상 | 처음 시작 | 코드 외부 반출 불가 환경 |

### 🅰️ 클라우드 방식

1. **OpenAI 키** 발급 → https://platform.openai.com/api-keys (`sk-`로 시작)
2. **Zilliz Cloud 키** 발급 → https://cloud.zilliz.com/signup (Personal API Key)
3. 등록:

```bash
claude mcp add claude-context \
  -e OPENAI_API_KEY=sk-your-openai-api-key \
  -e MILVUS_TOKEN=your-zilliz-cloud-api-key \
  -- npx @zilliz/claude-context-mcp@latest
```

> Zilliz Personal API Key를 쓰면 `MILVUS_ADDRESS`는 생략 가능 (토큰에서 자동 해석)

4. Claude Code에서 `/mcp` 입력 → `claude-context` 연결 확인

### 🅱️ 완전 로컬 방식 (무료)

```bash
# 1. Ollama 설치 후 임베딩 모델 받기
ollama pull nomic-embed-text

# 2. Milvus 로컬 설치 (Docker Compose)
#    https://milvus.io/docs/install_standalone-docker-compose.md

# 3. 등록
claude mcp add claude-context \
  -e EMBEDDING_PROVIDER=Ollama \
  -e OLLAMA_MODEL=nomic-embed-text \
  -e OLLAMA_HOST=http://127.0.0.1:11434 \
  -e MILVUS_ADDRESS=127.0.0.1:19530 \
  -- npx @zilliz/claude-context-mcp@latest
```

### 소스로 직접 빌드해서 쓰기

```bash
pnpm install
pnpm build

claude mcp add claude-context \
  -e OPENAI_API_KEY=sk-... \
  -e MILVUS_TOKEN=... \
  -- node /절대경로/packages/mcp/dist/index.js
```

> 커스터마이징할 게 아니면 `npx` 방식이 더 편하다 (최신 버전 자동).

---

## 4. 사용법

명령어를 외울 필요 없이 **자연어로 말하면** Claude가 알아서 도구를 호출한다.

```
① 이 코드베이스 인덱싱해줘        → index_codebase (백그라운드 실행)
② 인덱싱 상태 알려줘              → get_indexing_status
③ 로그인 처리하는 함수 찾아줘      → search_code  ⭐ 핵심
④ 인덱스 지워줘                   → clear_index
```

### 검색 예시

```
결제 관련 로직이 어디에 있어?
에러 핸들링하는 부분 다 보여줘
이 프로젝트에서 API 호출은 어떤 식으로 해?
사용자 인증이랑 비슷한 패턴 쓰는 코드 또 있어?
```

### 알아두면 좋은 것

- **여러 프로젝트 자동 구분**: 폴더를 옮겨다녀도 알아서 인식. 단 **절대경로 기준**이라 심볼릭 링크나 별도 클론은 다른 코드베이스로 취급된다.
- **증분 색인**: 두 번째부터는 머클 트리로 변경분만 재색인해 훨씬 빠르다.
- **`.gitignore` 자동 존중**: `node_modules` 등은 알아서 제외.
- 확장자 추가: `"인덱싱해줘, .vue랑 .svelte 파일도 포함해서"`
- 진행률이 초반에 10%로 급히 뛰는 것은 정상 동작이다.

### 설정 파일로 관리 (권장)

```bash
mkdir -p ~/.context
cat > ~/.context/.env << 'EOF'
EMBEDDING_PROVIDER=OpenAI
OPENAI_API_KEY=sk-your-key
EMBEDDING_MODEL=text-embedding-3-small
MILVUS_TOKEN=your-zilliz-key
EOF
```

이러면 등록이 간단해진다:

```bash
claude mcp add claude-context -- npx @zilliz/claude-context-mcp@latest
```

> ⚠️ 반드시 `~/.context/.env`에 둘 것. 프로젝트 폴더에 두면 프로젝트 환경변수와 충돌한다.

### 주요 환경변수

| 변수 | 설명 | 기본값 |
|---|---|---|
| `EMBEDDING_PROVIDER` | `OpenAI` / `VoyageAI` / `Gemini` / `Ollama` | `OpenAI` |
| `EMBEDDING_MODEL` | 모델명 (전 프로바이더 공통) | 프로바이더별 기본값 |
| `HYBRID_MODE` | 하이브리드 검색(BM25+벡터). `false`면 밀집벡터만 | `true` |
| `EMBEDDING_BATCH_SIZE` | 배치 크기. 클수록 색인 빠름 | `100` |
| `SPLITTER_TYPE` | `ast` / `langchain` | `ast` |
| `CUSTOM_EXTENSIONS` | 추가 확장자 (`.vue,.svelte`) | 없음 |
| `CUSTOM_IGNORE_PATTERNS` | 추가 제외 패턴 (`temp/**,*.backup`) | 없음 |

### 문제 해결 순서

1. `"인덱싱 상태 알려줘"` — 에러 메시지가 여기 다 나온다
2. `claude --debug` 로 로그 확인
3. **설정을 바꿨다면 반드시 재연결**: `/mcp reconnect claude-context`

---

## 5. 수익화 아이디어

### 먼저 냉정한 전제

- Zilliz가 이걸 무료로 배포하는 이유는 **Zilliz Cloud(벡터DB)를 팔기 위한 미끼상품**이기 때문이다. 본체로 정면승부하면 회사와 싸우는 셈.
- Cursor, GitHub Copilot은 이미 자체 코드 인덱싱을 내장했다. **개인 개발자용 코드 검색 툴 시장은 잠식 중.**
- 따라서 **돈은 본체가 아니라 "마찰이 생기는 지점"에 있다.**

### 🥇 1순위 — 팀 공유 인덱스 (B2B SaaS)

**발견한 빈틈**: 현재 구조는 철저히 개인용이다. 스냅샷이 `~/.context/merkle/`에 저장되고 절대경로로 키를 만든다.

> 팀원 10명 = 같은 코드를 10번 색인 = **임베딩 비용 10배.** 신입이 오면 처음부터 다시.

**제품**: 한 번만 색인해 팀 전체가 공유하는 인덱스 서버 + 권한 관리 / SSO / 감사 로그

**근거**: 개인은 무료 툴로 만족하지만, 기업은 "중복 비용 + 권한 관리"에 지불한다. 오픈소스가 건드리지 않는 영역.

**가격 모델**: 시트당 월 1~3만원 / 온프렘 연간 라이선스

### 🥈 2순위 — 규제 산업용 온프렘 패키지

**근거**: FAQ에 완전 로컬 배포(Ollama + 로컬 Milvus) 가이드가 있지만 설정이 번거롭다.

**제품**: "코드가 회사 밖으로 나가지 않는" 원클릭 설치 패키지

**대상**: 금융 / 공공 / 방산 — 망분리 환경이라 Cursor, Copilot을 **구조적으로 쓸 수 없다.** 경쟁자가 진입 불가능한 시장.

**가격 모델**: 구축비 + 연 유지보수 20% (전형적 SI 모델)

### 🥉 3순위 — 레거시 코드 분석 컨설팅

**대상**: 10년 된 SI 프로젝트, 문서 없음, 개발자 퇴사 → "이 기능 어디 있는지 아무도 모름" 상태

**제품**: 도구 + 사람. 이 툴로 레거시를 훑어 아키텍처 문서 / 의존성 맵을 산출물로 판매

**장점**: 자본 없이 즉시 시작 가능. 단 시간을 파는 구조라 확장성은 낮다. → **여기서 번 돈과 고객 데이터로 1·2순위 제품을 만드는 게 정석 루트.**

### 4순위 — 이걸 "부품"으로 쓰는 다른 제품

| 아이디어 | 설명 |
|---|---|
| 온보딩 봇 | 신입이 "결제 로직 어디예요?" 물으면 답하는 슬랙 봇 |
| 자동 문서 생성 | 코드를 훑어 아키텍처 문서 생성 |
| **중복코드 탐지** | 의미 기반이라 이름이 달라도 잡아낸다 |
| 보안 패턴 감사 | "인증 없이 DB에 접근하는 코드"를 의미로 탐색 |

> 중복코드 탐지가 특히 유망하다. 기존 툴은 문자열 비교라 `getUserData`와 `fetchMemberInfo`가 같은 일을 하는 걸 못 잡지만, 이건 잡는다.

### 5순위 — 콘텐츠

MCP 서버 제작 강의, AI 코딩 도구 뉴스레터, 한국어 세팅 가이드 블로그. 리스크 0, 즉시 시작 가능하나 수익 규모는 작다. **인지도 축적용으로 병행.**

### 추천 실행 루트

```
1단계 (~3개월)    3순위 컨설팅 → 현금 확보 + 실제 고객 문제 파악
2단계 (3~9개월)   1순위 팀 공유 인덱스 MVP
3단계 (9개월~)    2순위 온프렘 패키지로 B2B 확장
```

**핵심 원칙**

> ❌ 오픈소스와 **같은 걸** 더 잘 만들려 하지 말 것  
> ✅ 오픈소스가 **안 하는 걸** 할 것 (팀 · 권한 · 규제 · 서비스)

> ⚠️ 위 내용은 검증된 사업 계획이 아니라 **가설**이다. 실제 고객 5명에게 확인해야 유효성을 알 수 있다.

---

## 6. PHP로 만들 수 있는가

### 결론: 가능하다. 단 "전부 다"는 하지 말 것.

### ✅ 가능한 이유 (결정적 근거)

`packages/core/src/vectordb/milvus-restful-vectordb.ts` 파일의 존재.

```
주석: "gRPC가 불가능한 제한된 환경(VSCode/Chrome 확장)을 위해 HTTP 요청만으로 구현"

baseUrl = 주소 + '/v2/vectordb'
POST /collections/create, /entities/insert, /entities/search, /entities/hybrid_search ...
```

**Milvus를 전부 HTTP API로 다룰 수 있다.** PHP SDK가 없어도 Guzzle로 충분하다. 임베딩(OpenAI)도 HTTP다. → PHP의 최대 걸림돌로 예상했던 부분이 이미 해결돼 있다.

### 부분별 난이도 (실제 코드량 기준)

| 기능 | 코드량 | PHP 난이도 | 판정 |
|---|---|---|---|
| 벡터DB 연동 | REST API | 쉬움 | ✅ Guzzle |
| 임베딩 호출 | HTTP POST | 쉬움 | ✅ |
| 머클트리 증분 | 89줄 | 쉬움 | ✅ `hash_file()` |
| 문자 기반 청킹 | 131줄 | 쉬움 | ✅ 단순 포팅 |
| gitignore 처리 | 라이브러리 | 보통 | ✅ PHP 패키지 존재 |
| **AST 파싱** | 270줄 + tree-sitter | 어려움 | ⚠️ 아래 참고 |
| **MCP 서버** | stdio 방식 | 애매 | 🚫 만들지 말 것 |

### ⚠️ 걸림돌 1 — tree-sitter

tree-sitter는 C 라이브러리로, Node는 네이티브 바인딩이 성숙하지만 PHP는 FFI로 붙여야 해 불리하다.

**대안 3가지**

1. 문자 기반 청킹만 사용 (품질 소폭 하락, 원본에도 폴백으로 존재하는 기능)
2. `tree-sitter-cli`를 외부 프로세스로 호출 (이미 이 저장소 devDependency에 있음)
3. **PHP 코드가 대상이라면 `nikic/PHP-Parser` 사용** ← 최선

### 🚫 걸림돌 2 — MCP 서버는 PHP로 만들지 말 것

기술이 아니라 **배포 문제**다.

```
Node 버전 → npx 한 줄. 개발자 대부분 이미 설치돼 있음     ✅
PHP 버전 → "PHP CLI 8.2를 설치하세요"부터 시작            ❌
```

MCP는 개발자 PC에서 도는 클라이언트라 **설치 마찰이 곧 사망**이다. 게다가 원본이 MIT이므로 **그냥 재활용하면 된다.**

### 권장 아키텍처

```
┌─────────────────────────────────────┐
│  개발자 PC                            │
│  기존 Node MCP 그대로 사용 (MIT)       │  ← 만들지 말고 재활용
└──────────────┬──────────────────────┘
               │ HTTP
               ▼
┌─────────────────────────────────────┐
│  우리 서버 = Laravel (PHP)            │  ← 여기가 제품
│  · 팀 / 권한 / SSO / 결제 / 대시보드   │
│  · 감사 로그                          │
│  · 인덱싱 잡 큐 (Horizon)             │
└──────────────┬──────────────────────┘
               │ REST
               ▼
        Milvus + 임베딩 API
```

**핵심**: 1순위 "팀 공유 인덱스" 제품의 실제 내용물은 결국 계정 · 권한 · 조직 관리 · 결제 · 대시보드 · 감사 로그 · 잡 큐다. **전부 Laravel이 가장 잘하는 영역이므로 PHP 선택이 오히려 유리하다.**

### 💎 PHP만의 기회 — "PHP 레거시 전문 시맨틱 검색"

1. 국내에 PHP 레거시가 매우 많다 (그누보드, 구형 SI, 워드프레스 커스텀 등)
2. **`nikic/PHP-Parser`는 tree-sitter보다 PHP를 정확하게 파싱한다** (PHP 생태계 표준)
3. **원본은 PHP를 문자 단위로만 자른다** — AST 지원 목록에 PHP가 없다

→ **원본이 못 하는 걸 우리가 더 잘하는 유일한 지점.** "PHP 코드는 PHP 도구로 분석한다"는 명분까지 확보된다. 3순위 컨설팅 아이디어와도 자연스럽게 연결된다.

### PHP 구현 시 주의사항

| 함정 | 대응 |
|---|---|
| 색인은 오래 걸림 → PHP-FPM 타임아웃 | 반드시 큐 워커로 (Laravel Horizon) |
| 대용량 파일 메모리 초과 | 스트리밍 처리 + `memory_limit` 조정 |
| 해싱/파싱은 CPU 바운드 → Node보다 느릴 수 있음 | 워커 프로세스 수로 보완 |
| 임베딩 API 배치 | 원본과 동일하게 100개 단위 |

### 최종 판정

| 부분 | PHP로? |
|---|---|
| MCP 클라이언트 | ❌ 기존 Node 재활용 (MIT라 합법) |
| 서버 / SaaS 레이어 | ✅✅ Laravel이 최적 |
| 인덱싱 워커 | ✅ 가능 (REST라 전부 됨) |
| PHP 코드 AST 분석 | ✅✅✅ PHP가 오히려 최강 |
| 그 외 언어 AST | ⚠️ 문자 청킹 또는 CLI 호출 |

> **바퀴를 다시 만들지 말고, 바퀴 위에 우리 차를 얹는다.**  
> Node 엔진은 재활용하고, 돈이 되는 부분(팀 · 권한 · 결제 · PHP 특화)만 Laravel로 만든다.

---

## 7. 다음 액션 후보

- [ ] `nikic/PHP-Parser` 기반 PHP 코드 청킹 프로토타입 (1일 규모)
- [ ] Laravel + Milvus REST 연동 PoC
- [ ] 잠재 고객 5명 인터뷰로 1순위 가설 검증
- [ ] 로컬 무료 구성(Ollama + Docker Milvus)으로 실제 색인 테스트
