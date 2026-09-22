# Cursor Plugins 레포 전수조사 분석 정리 (한국어)

> 작성일: 2026-09-22
> 분석 대상 레포: <https://github.com/bmshin94/plugins>
> 원본(업스트림) 레포: <https://github.com/cursor/plugins>
> 분석 도구: Claude Code

---

## 0. 가장 중요한 정정 사항

레포 루트의 `CLAUDE.md`에는 이 프로젝트가 **"openai/plugins — ChatGPT 공식 플러그인 표준 규격"**
이라고 적혀 있으나, **이는 사실이 아니다.** (자동 생성 문서의 오류)

실제 파일을 전수조사한 결과 다음과 같이 확인되었다.

| 항목 | 실제 확인 내용 | 근거 파일 |
|:--|:--|:--|
| 정체 | **Cursor 에디터 공식 플러그인 마켓플레이스** | `README.md` 1행 `# Cursor plugins` |
| 매니페스트 | `.cursor-plugin/plugin.json` | 전 플러그인 공통 |
| 스키마 `$id` | `https://cursor.com/schemas/cursor-plugin/plugin.json` | `schemas/plugin.schema.json` |
| 지원 클라이언트 | `cursor`, `grokbot`(Grok Bot), 구식 별칭 `sand` | `schemas/plugin.schema.json` |
| 원본 | `cursor/plugins` (bmshin94/plugins는 이를 복제) | `third_party/*/plugin.json` 의 `repository` |
| 라이선스 | MIT | `README.md`, 각 플러그인 `LICENSE` |

OpenAI의 ChatGPT 플러그인 규격은 이미 GPTs/Actions로 대체되어 폐기되었고,
이 레포는 그것과 **아무 관련이 없다.**

---

## 1. 레포 전체 구조

```
plugins/                              # 파일 769개 / 최상위 폴더 23개
├── .cursor-plugin/marketplace.json   # 마켓 전체 카탈로그 (플러그인 79개 등재)
├── schemas/
│   ├── marketplace.schema.json       # 마켓 매니페스트 규격
│   └── plugin.schema.json            # 플러그인 매니페스트 규격 (핵심 자산)
├── scripts/validate-plugins.mjs      # ajv 기반 스키마 검증 스크립트
├── .github/workflows/validate-plugins.yml  # PR마다 자동 검증 CI
├── [1st-party 플러그인 15개]         # AI 작업 방식을 바꾸는 "워크플로우형"
└── third_party/ (64개)               # 외부 SaaS 연동 "MCP형"
```

**구성 요소 총량**

| 구성요소 | 개수 |
|:--|--:|
| SKILL.md (스킬) | 92 |
| agents/*.md (서브에이전트) | 13 |
| mcp.json (MCP 서버 정의) | 62 |
| rules/*.mdc (룰) | 3 |
| hooks (훅 세트) | 3 세트 (advisor, ralph-loop, continual-learning) |

---

## 2. 플러그인 2분류

### A. 1st-party 15개 — API 키 불필요, 마크다운이 본체

| 플러그인 | 설명 | 특징 |
|:--|:--|:--|
| `pstack` | 개발 방법론 OS (Lauren Tan) | **스킬 47개**, `principle-*` 35종, TDD·근본원인분석·병렬 서브에이전트 |
| `thermos` | "열핵 리뷰" 보안+품질 동시 감사 | 서브에이전트 2개 병렬 실행 후 결과 합성 |
| `advisor` | 더 강한 모델에게 자문 | `model: grok-4.6[effort=xhigh]`, `readonly: true` |
| `orchestrate` | 대형 작업 병렬 분산 | planner / worker / verifier 구조 |
| `ralph-loop` | 자기 반복 루프 | `stop` 훅 가로채기 (`loop_limit: null`) |
| `continual-learning` | 장기 기억 | 세션 종료 시 `AGENTS.md` 자동 갱신 (TS 훅) |
| `create-plugin` | 플러그인 생성기 (메타) | 스캐폴딩 + 제출 리뷰 스킬 |
| `cursor-team-kit` | 팀 CI/리뷰/배포 워크플로 | 룰 2종 포함 |
| `agent-compatibility` | 레포의 에이전트 친화도 스캔 | |
| `cli-for-agent` | 에이전트 친화적 CLI 설계 패턴 | |
| `pr-review-canvas` | PR diff 시각 캔버스 렌더링 | |
| `docs-canvas` | 문서 캔버스 렌더링 | |
| `cursor-sdk` | TypeScript SDK 자동화 | |
| `grok-voice` | 실시간 음성(STT/TTS) 연동 | |
| `teaching` | 학습 로드맵·회고 | |

### B. third_party 64개 — 외부 SaaS 연결(MCP 래퍼)

Gmail, Google Drive/Calendar/BigQuery, GitHub, Teams, SharePoint, Salesforce, HubSpot,
Playwright, Coda, Craft, Mem, Readwise, Fireflies, Otter, Fathom, Circleback,
Brex, Mercury, Xero, Webull, Interactive Brokers, S&P Global, Daloopa,
Klaviyo, Brevo, Customer.io, MailerLite, Semrush, Ahrefs, Similarweb,
Ashby, Workable, Juicebox, Upwork, Outreach, Amplemarket, Clay, Hunter, Attio,
Typeform, Jotform, Smartsheet, Wrike, Todoist, Calendly, Docusign, Navan,
Gong, Intercom, Zoom, Guru, Gamma, Excalidraw, Profound, Meltwater, GoDaddy,
X, X Ads, Finance, OneDrive, Outlook, Outlook Calendar 등

핵심은 **코드가 거의 없다는 점**이다. 예: GitHub 플러그인 전체 실체

```jsonc
// third_party/github/mcp.json
{ "mcpServers": { "github": {
    "type": "http",
    "url": "https://api.githubcopilot.com/mcp/",
    "headers": { "Authorization": "Bearer ${GITHUB_PERSONAL_ACCESS_TOKEN}" }
}}}
```

서버 구현체는 각 SaaS가 자체 호스팅하고, 이 레포는 **"주소록 + 설치 설명서"** 역할만 한다.

---

## 3. 플러그인 규격 (schemas/plugin.schema.json)

| 필드 | 의미 |
|:--|:--|
| `name` | kebab-case 고유 식별자 (필수) |
| `skills` | SKILL.md 경로/글롭 |
| `agents` | 서브에이전트 정의 |
| `rules` | `.mdc` 상시 적용 규칙 |
| `commands` | 슬래시 명령어 |
| `hooks` | `afterAgentResponse`, `stop` 등 생명주기 훅 |
| `mcpServers` | MCP 서버 (경로/인라인/배열) |
| `variables` | 사용자 입력값(토큰) JSON Schema |
| `minClientVersions` | 클라이언트별 최소 버전 또는 `"never"` (숨김) |

---

## 4. 쉬운 비유 정리

AI를 **신입사원**이라고 보면:

| 구성요소 | 비유 | 적용 시점 |
|:--|:--|:--|
| Skill | 업무 매뉴얼 | 필요할 때 꺼내 읽음 |
| Rule | 사내 취업규칙 | 항상 적용 |
| Agent | 외부 전문가 호출 | 위임 |
| Hook | 자동 알람 | 특정 시점 자동 실행 |
| MCP | 사내 시스템 계정(열쇠) | 외부 연결 |
| Plugin | 위를 담은 온보딩 키트 상자 | 배포 단위 |
| 이 레포 | 상자 창고 + 카탈로그 | 마켓플레이스 |

| | 1st-party | third_party |
|:--|:--|:--|
| 내용물 | 글(마크다운) | 주소+열쇠(JSON) |
| 역할 | AI의 *사고방식* 개조 | AI에 *손발* 달기 |
| 인증 | 불필요 | 토큰/OAuth |

> 핵심 통찰: 일반 오픈소스는 코드가 자산이지만,
> 이 레포는 **"AI에게 하는 말"이 자산**이다. = 프롬프트가 곧 소프트웨어.

---

## 5. 자주 묻는 질문 (Q&A)

### Q1. 설치 및 사용법

**방법 A — Cursor 사용자**
```bash
/add-plugin pstack
/add-plugin thermos
/add-plugin github
```

**방법 B — 로컬 커스텀**
```bash
git clone https://github.com/bmshin94/plugins
cp -r plugins/thermos ~/.cursor/plugins/local/thermos
```
(`create-plugin/rules/plugin-quality-gates.mdc`에 기본 저장 경로가 명시되어 있음)

**방법 C — Cursor 없이 Claude Code로 재활용 (추천)**
```bash
mkdir -p ~/.claude/skills
cp -r plugins/pstack/skills/tdd ~/.claude/skills/
```
SKILL.md 포맷(YAML frontmatter + 마크다운)이 사실상 호환된다.
`disable-model-invocation` 등 일부 Cursor 전용 필드만 무시된다.

**검증**
```bash
npm install --no-save ajv ajv-formats
node scripts/validate-plugins.mjs
```

### Q2. 플러그인? 스킬? MCP?

**셋 다 맞다.** "플러그인"이 나머지를 담는 최상위 그릇이다.

```
플러그인 (배포 단위)
  ├─ skills/   ← 스킬 (AI가 읽는 지침 문서)
  ├─ agents/   ← 서브에이전트
  ├─ rules/    ← 룰
  ├─ hooks/    ← 훅
  └─ mcp.json  ← MCP (AI↔외부API 통신 프로토콜)
```

- `pstack`, `thermos` → 스킬 전용 (MCP 없음)
- `gmail`, `github` → MCP 전용 (스킬 없음)
- `x`, `juicebox` → 하이브리드

### Q3. API 토큰이 필요한가?

| 구분 | 개수 | 토큰 |
|:--|--:|:--|
| 1st-party | 15 | 불필요 (마크다운일 뿐) |
| third_party OAuth형 | 52 | 직접 입력 불필요 (클릭 로그인) |
| third_party 토큰형 | 12 | 직접 발급·입력 필요 |

직접 입력이 필요한 사례: GitHub(PAT), Hunter, Similarweb, Smartsheet, Wrike,
Brevo, Xero(CLIENT_ID + CLIENT_SECRET) 등.
전체에서 `CLIENT_ID` 10건, `CLIENT_SECRET` 8건이 선언되어 있다.

전송 방식: `http` 61개(원격), `stdio` 1개(xero, `npx @xeroapi/xero-mcp-server`).

**보안 주의**: 토큰을 `mcp.json`에 하드코딩하면 안 되고 반드시 `${VAR}` 치환을 써야 한다.

### Q4. 왜 GitHub에서 유명한가 (레포 내용 기반 추론)

1. Cursor 공식 레포 + Cursor가 현재 가장 널리 쓰이는 AI 에디터
2. **"프롬프트 오픈소스"의 희소성** — 보통 영업비밀인 프롬프트 92개를 MIT로 공개
3. **긁어가기 쉬움** — 파일 복사만으로 Claude Code/Copilot에도 적용 가능
4. 64개 SaaS의 MCP 서버 URL 카탈로그 자체가 레퍼런스 가치
5. 네트워크 효과 — 기업들이 자사 통합을 PR로 기여 (SharePoint/Finance, X Chat 등 실제 커밋 확인)
6. `thermos`, `ralph-loop`, `poteto-mode` 등 밈성 작명의 바이럴 효과
7. 진입장벽 제로 — JSON 6줄이면 플러그인 하나 완성

### Q5. 로컬 에이전트 구축에 도움이 되는가 → 매우 도움 됨

| 가져다 쓸 것 | 위치 |
|:--|:--|
| 스킬 설계 패턴 | 92개 `SKILL.md` |
| MCP 서버 주소록 | `third_party/*/mcp.json` 62개 |
| 서브에이전트 오케스트레이션 | `thermos/skills/thermos/SKILL.md` |
| 자기개선 루프 | `ralph-loop/hooks/hooks.json` |
| 장기 기억 | `continual-learning/hooks/*.ts` |
| 강한 모델 자문 패턴 | `advisor/agents/advisor-subagent.md` |
| 플러그인 시스템 설계 | `schemas/*.json` + `scripts/validate-plugins.mjs` |

**한계**
- 에이전트 **런타임(실행엔진)은 없다.** 툴콜 루프·컨텍스트 관리는 직접 구현해야 함
- Cursor 전용 필드(`minClientVersions`, `.mdc`)는 타 클라이언트와 비호환
- 훅은 Cursor 생명주기 이벤트에 의존 → 재구현 필요

### Q6. React나 PHP로 만들 수 있는가

**플러그인 자체는 JSON + 마크다운**이라 언어가 낄 자리가 없다.
그러나 **MCP 서버는 PHP로 100% 만들 수 있다.** MCP는 JSON-RPC 2.0 over HTTP이기 때문이다.

| 레이어 | 기술 | React/PHP |
|:--|:--|:--|
| 플러그인 매니페스트 | JSON | 언어 무관 |
| 스킬/룰 | Markdown | 언어 무관 (한글 가능) |
| 훅 | Bash / TS | PHP CLI 가능 |
| **MCP 서버** | 자유 | **PHP 가능** |
| 마켓 웹 UI | 자유 | **React 가능** |

**PHP MCP 서버 뼈대**

```php
<?php // mcp.php
header('Content-Type: application/json');
$req = json_decode(file_get_contents('php://input'), true);

switch ($req['method']) {
  case 'initialize':
    $result = ['protocolVersion' => '2025-06-18',
               'capabilities' => ['tools' => new stdClass()],
               'serverInfo' => ['name' => 'my-php-mcp', 'version' => '1.0.0']];
    break;
  case 'tools/list':
    $result = ['tools' => [[
      'name' => 'get_order',
      'description' => '주문번호로 주문 상세 조회',
      'inputSchema' => ['type' => 'object',
        'properties' => ['order_id' => ['type' => 'string']],
        'required' => ['order_id']]
    ]]];
    break;
  case 'tools/call':
    $id = $req['params']['arguments']['order_id'];
    $result = ['content' => [['type' => 'text', 'text' => "주문 $id: 배송중"]]];
    break;
}
echo json_encode(['jsonrpc' => '2.0', 'id' => $req['id'], 'result' => $result]);
```

**플러그인 포장**

```jsonc
// my-shop/mcp.json
{ "mcpServers": { "my-shop": {
    "type": "http", "url": "https://myshop.co.kr/mcp.php",
    "headers": { "Authorization": "Bearer ${MYSHOP_API_KEY}" }
}}}
```
```jsonc
// my-shop/.cursor-plugin/plugin.json
{ "name": "my-shop", "displayName": "우리쇼핑몰", "version": "1.0.0",
  "description": "주문·재고·정산 조회",
  "variables": { "type": "object",
    "properties": { "MYSHOP_API_KEY": { "type": "string", "title": "API 키" } },
    "required": ["MYSHOP_API_KEY"] },
  "mcpServers": "./mcp.json" }
```

React 활용처: 플러그인 마켓 UI, 스킬 작성 GUI 에디터, MCP 호출 로그 대시보드, 캔버스 렌더러.

---

## 6. 수익화 아이디어

### 티어 1 — 즉시 실행 (1~4주, 저비용)

**① 한국형 MCP 플러그인 팩 "K-Plugins"** (최우선 추천)
- 갭: 등재된 79개 중 **한국 서비스 0개**. 네이버·카카오·토스·쿠팡·배민·더존 전부 공백
- 구현: PHP로 MCP 서버 작성 → 플러그인 포장
- 모델: 기본 무료 / Pro $9월 / 팀 $49월
- 이점: `cursor/plugins`에 PR 등재 시 공식 마켓 노출 = 무료 마케팅
- 예상: 1000 MAU × 3% × $9 ≈ 월 $270~1,000

**② MCP 브릿지 SaaS — 레거시를 AI로** (수익성 최고)
- 타깃: PHP 기반 한국 중소기업 ERP/쇼핑몰/그누보드
- 제품: REST API/DB → MCP 자동 변환 어댑터 + React 관리자
- 가격: 셀프 $29월 / 비즈 $99월 / 엔터프라이즈 $499월 + 구축비 300~1000만원
- 예상: 고객 30곳 × $99 ≈ 월 $3,000 + 구축비

**③ 스킬 번들 유료 판매**
- 상품: "한국 개발팀용 AI 워크플로 30종"
- 가격: $29~79 일회성 / $19월 구독
- 주의: MIT라 참고·재구성은 자유지만 **저작권 고지 유지 의무** 준수
- 예상: 월 20~50건 × $39 ≈ 월 $780~2,000

### 티어 2 — 3~6개월 (중간 투자)

**④ 사내 플러그인 마켓플레이스 (B2B 온프렘)** — 최대 매출
- 근거: `marketplace.json` + `schemas/` + `validate-plugins.mjs` 구조가 그대로 제품 설계도
- 제품: React 대시보드 + 사내 레지스트리 + SSO 권한관리 + 감사로그 + 검증 CI
- 가격: 연 2,000~5,000만원 / 예상: 고객 5곳 ≈ 연 1~2.5억

**⑤ MCP 게이트웨이 / 관측 플랫폼**
- 근거: 현재 62개 서버가 **날것의 토큰**을 클라이언트에 직접 주입 → 기업 보안팀 승인 불가
- 제품: 토큰 금고, 요청 로깅, 레이트리밋, 비용 추적, PII 마스킹 프록시
- 가격: $0.001/호출 + 기본료 $99월 / 예상: 월 $5,000~20,000

**⑥ 스킬 작성 GUI 에디터 (React)**
- 폼 입력 → SKILL.md/plugin.json 생성 + 실시간 스키마 검증 + 원클릭 PR
- 무료 + Pro $15월, 마켓 수수료 15~30%

### 티어 3 — 교육·콘텐츠 (현금흐름 빠름)

**⑦ 온라인 강의** — 8~12시간, 8~15만원, 300~1000명 → 2,400만~1.5억
**⑧ 기업 출강·컨설팅** — 워크샵 회당 300~800만원, 구축 1,000~3,000만원
**⑨ 뉴스레터/유료 커뮤니티** — 월 $19 + 스폰서십 건당 $500~2,000

### 종합 로드맵

```
0~1개월    ③ 스킬 번들 + ⑨ 뉴스레터   → 현금흐름 + 신뢰 구축
1~3개월    ① K-Plugins 오픈소스 공개   → 공식 마켓 PR = 인지도
3~6개월    ② MCP 브릿지 SaaS           → ①의 리드 전환
6~12개월   ④ 사내 마켓플레이스 B2B     → ②고객사 업셀 = 최대 매출
병행       ⑦ 강의 + ⑧ 컨설팅          → 권위의 현금화
```

**PHP 백엔드(MCP 서버) + React 프론트(관리자)** 조합에 정확히 부합하는 전략이다.

### 리스크

- MCP 스펙 변동성 → 버전 고정 및 마이그레이션 자동화 필요
- Cursor/Anthropic의 유사 기능 직접 출시 가능성 → "한국 특화"가 방어선
- 토큰 보관은 법적 책임 (개인정보보호법) → 암호화·최소수집 원칙
- MIT 라이선스: 상업 이용 자유이나 저작권 고지 유지 의무

---

## 7. 참고 링크

| 항목 | URL |
|:--|:--|
| 본 레포 | <https://github.com/bmshin94/plugins> |
| 원본(업스트림) | <https://github.com/cursor/plugins> |
| 플러그인 스키마 | `schemas/plugin.schema.json` |
| 마켓 매니페스트 | `.cursor-plugin/marketplace.json` |
| 검증 스크립트 | `scripts/validate-plugins.mjs` |
| pstack (스킬 47개) | `pstack/skills/` |
| MCP 연동 예시 | `third_party/github/mcp.json` |

---

## 8. 한 줄 결론

> 이 레포는 **ChatGPT 플러그인 규격이 아니라 Cursor 에디터용 AI 에이전트 확장 마켓플레이스**이며,
> 실질 가치는 코드가 아니라 **92개의 공개된 고품질 프롬프트(SKILL.md)와 62개 MCP 연동 카탈로그**에 있다.
> 한국 서비스 연동이 전무하다는 공백이 가장 명확한 사업 기회다.
