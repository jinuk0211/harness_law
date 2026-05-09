최근 클로드 코드, 코덱스 잘 쓰는법 skill,plugin,mcp 활용 같은 글들 bottom으로 내려가 baseline 코드 분석

https://arxiv.org/html/2604.14228v1
openclaw와 claude code, 최근의 hermes와 같이 단순히 tool calling, mcp를 넘어 하네스를 통해 agentic workflow를 구축하는 데 그 중 하나인 claude code에서의 아키텍쳐에 대해 소개해놓은 논문

클로드 코드가 무엇인지에 대한 안트로픽 공식문서 묘사
ReAcT 형식으로 계획을 수립하고 액션(파일 읽기, 쓰기, shell 커맨드 실행)을 취하고 이를 task가 완료될때까지 수행하는 agentic loop

다루기 앞서 클로드 코드에 관한 근본적인 질문과 대답
===============================================
1. 추론(reasoning) 어디에 존재하는가? 
Claude Code는 AI가 “무엇을 할지”만 결정하고, 실제 실행은 하네스(harness)가 담당 = response의 tool_use 블럭 안의 지시
모델은 직접 파일·쉘·네트워크에 접근하지 못하며 tool_use 요청만 보낸다.
그래서 모델이 이상 행동을 해도 권한 제한과 샌드박스를 우회할 수 없다.

2. 실행 엔진은 몇 개인가?
Claude Code는 CLI, IDE, SDK 등 모든 환경에서 하나의 queryLoop() 실행 엔진을 사용한다. = query.ts
달라지는 것은 UI와 렌더링 계층뿐이다.
덕분에 동작 일관성과 유지보수성이 높다.

3. 기본 안전 정책은 무엇인가?
기본 철학은 “우선 거부(Deny-first), 필요하면 사용자에게 확인 = 클로드 코드 사용시 나오는 허락
deny > ask > allow 순서로 규칙이 적용된다.
권한 규칙·샌드박스·훅 등 여러 안전장치가 동시에 작동한다.

4. 가장 중요한 자원 제약은 무엇인가?
Claude Code의 핵심 제약은 연산량보다 컨텍스트 윈도우 크기다. = CLAUDE.md, skill.md, agent.md, plugin, rule.md, memory.md
그래서 모델 호출 전마다 여러 단계의 컨텍스트 압축과 요약을 수행한다.
짧은 압축부터 의미 기반 요약까지 단계적으로 적용해 토큰 사용량을 줄인다.

<img width="3855" height="1405" alt="image" src="https://github.com/user-attachments/assets/25c9a532-da10-4e08-9b03-b707fc8deff2" />
클로드 코드의 구성요소

1. User
사용자는 프롬프트 입력, 권한 승인, 결과 검토를 담당한다.
중요 작업은 사용자 확인을 거쳐야 한다.
(별도 ts 파일보다는 전체 시스템 상위 개념 역할)

2. Interfaces
CLI, SDK, IDE, 브라우저 등 여러 인터페이스가 존재한다.
모든 인터페이스는 동일한 내부 루프로 연결된다.
즉 UI만 다르고 핵심 실행 구조는 같다.

3. Agent loop — query.ts
핵심 실행 흐름은 query.ts의 queryLoop()에서 구현된다.
모델 호출 → tool 실행 → 결과 수집을 반복한다.
Claude Code orchestration의 중심 엔진이다.

4. Permission system — permissions.ts, types/hooks.ts
권한 평가는 permissions.ts에서 deny-first 방식으로 처리된다.
types/hooks.ts는 hook 기반 interception을 담당한다.
위험 행동은 차단하거나 사용자 승인을 요구한다.

5. Tools — tools.ts
도구 조합은 tools.ts의 assembleToolPool()이 담당한다.
최대 54개 내장 도구와 MCP 도구를 통합한다.
실제 에이전트 기능 대부분이 여기서 제공된다.

6. State & persistence — sessionStorage.ts, history.ts
sessionStorage.ts는 JSONL 세션 기록 저장을 담당한다.
history.ts는 전역 프롬프트 히스토리를 관리한다.
장기 상태와 작업 맥락 유지 역할을 한다.

7. Execution environment — shouldUseSandbox.ts
shouldUseSandbox.ts는 샌드박스 사용 여부를 결정한다.
쉘 실행, 파일 접근, 웹 요청 등이 이 계층에서 수행된다.
즉 외부 세계와 실제 연결되는 실행 환경이다.

<img width="793" height="291" alt="image" src="https://github.com/user-attachments/assets/72c05446-6069-47b1-ba74-02e344d642bc" />

### 1. Surface layer — `src/entrypoints/`, `src/screens/`, `src/components/`

사용자 인터페이스 계층이다.
`src/entrypoints/`는 CLI·SDK 시작점, `src/screens/`는 전체 화면 구성, `src/components/`는 terminal UI 컴포넌트를 담당한다.
모든 인터페이스는 같은 agent loop로 연결되며 UI만 다르다.

### 2. Core layer — `query.ts`

핵심 agent loop는 `query.ts`의 `queryLoop()`에서 구현된다.
모델 호출 전마다 budget reduction, snip, microcompact 등 5단계 context compaction을 수행한다.
즉 Claude Code의 orchestration과 context 관리 중심 계층이다.

### 3. Safety/action layer — `permissions.ts`, `types/hooks.ts`, `tools.ts`, `shouldUseSandbox.ts`

권한 시스템은 `permissions.ts`에서 deny-first 정책으로 동작한다.
`types/hooks.ts`는 hook interception, `tools.ts`는 tool pool 조립, `shouldUseSandbox.ts`는 shell sandbox를 담당한다.
subagent도 일반 tool처럼 실행되며 독립 context를 사용한다.

### 4. State layer — `context.ts`, `sessionStorage.ts`, `history.ts`, `claudemd.ts`

`context.ts`는 system/user context를 조립한다.
`sessionStorage.ts`는 JSONL 세션 저장, `history.ts`는 global history, `claudemd.ts`는 instruction hierarchy를 관리한다.
subagent 대화도 별도 sidechain 파일로 저장해 parent context 오염을 막는다.

### 5. Backend layer — `BashTool.tsx`, `PowerShellTool.tsx`, `services/mcp/client.ts`

실제 shell 실행, 원격 실행, MCP 연결 등을 담당하는 계층이다.
MCP는 stdio·HTTP·WebSocket 등 다양한 transport를 지원한다.
`src/tools/` 아래 실제 tool 로직들이 구현된다.

---

### 6. QueryEngine — `QueryEngine.ts`

`QueryEngine`은 핵심 엔진이 아니라 비대화형 환경용 conversation wrapper다.
실제 공통 실행 경로는 `query.ts`의 `query()`와 `queryLoop()`이다.
interactive CLI는 QueryEngine 없이 바로 query()를 호출한다.

---

### 7. 7중 Safety Layer

Claude Code는 7개의 독립 안전 계층을 사용한다.
tool pre-filtering → deny-first rules → permission mode → ML classifier → sandbox → permission reset → hooks 순으로 방어한다.
하나라도 차단하면 실행되지 않는 defense-in-depth 구조다.

---

### 8. Context Bottleneck 최적화

Claude Code는 context window를 가장 큰 병목으로 본다.
그래서 lazy loading, deferred tool schema, subagent summary-only return 등을 사용한다.
즉 “필요한 정보만 context에 넣는 것”이 핵심 설계 철학이다.
