실제 클로드 코드 분석
================================================
LLM을 둘러싸고 있는 하네스에 관해 - https://arxiv.org/html/2604.14228v1

openclaw와 claude code, 최근의 hermes와 같이 단순히 tool calling, mcp를 넘어 하네스를 통해 agentic workflow를 구축해 놓은 프로젝트가 많은데 그 중 하나인 claude code에서의 아키텍쳐에 대해 소개해놓은 논문

클로드 코드가 무엇인지에 대한 안트로픽 공식문서 묘사 =
ReAcT 형식으로 계획을 수립하고 액션(파일 읽기, 쓰기, shell 커맨드 실행)을 취하고 이를 task가 완료될때까지 수행하는 agentic loop

다루기 앞서 클로드 코드에 관한 근본적인 질문과 대답
===============================================
1. 추론은(reasoning) 어디에 존재하는가? 
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
-------------------
1. User
사용자는 프롬프트 입력, 권한 승인, 결과 검토를 담당
중요 작업은 사용자 확인을 거쳐야 한다.
(별도 typescript 파일이라기 보다는 전체 시스템 상위 개념 역할)

2. Interfaces
CLI, SDK, IDE, 브라우저 등 여러 인터페이스가 존재
모든 인터페이스는 동일한 내부 루프로 연결
즉 UI만 다르고 핵심 실행 구조는 같다

3. Agent loop — query.ts
핵심 실행 흐름은 query.ts의 queryLoop()에서 실행
모델 호출 → tool 실행 → 결과 수집을 반복
Claude Code orchestration의 핵심

4. Permission system — permissions.ts, types/hooks.ts
권한 평가는 permissions.ts에서 deny-first 방식으로 처리
types/hooks.ts는 hook 기반 interception을 담당
위험 행동은 차단하거나 사용자 승인을 요구한다.

5. Tools — tools.ts
도구 조합은 tools.ts의 assembleToolPool()이 실행
최대 54개 내장 도구와 MCP 도구를 통합
실제 에이전트 기능 대부분이 여기서 제공

6. State & persistence — sessionStorage.ts, history.ts
sessionStorage.ts는 JSONL 세션 기록 저장을 담당
history.ts는 global 프롬프트 히스토리를 관리
장기 상태와 작업 맥락 유지 역할을 한다.

7. Execution environment — shouldUseSandbox.ts
shouldUseSandbox.ts는 샌드박스 사용 여부를 결정
쉘 실행, 파일 접근, 웹 요청 fetch, MCP 호출 등이 이 계층에서 수행됨
즉 외부환경과 실제 연결되는 실행 환경

<img width="793" height="291" alt="image" src="https://github.com/user-attachments/assets/72c05446-6069-47b1-ba74-02e344d642bc" />

Layered Subsystem Decomposition
------------------
### 1. Surface layer — `src/entrypoints/`, `src/screens/`, `src/components/`

사용자 인터페이스 계층으로
`src/entrypoints/`는 CLI·SDK 시작점, `src/screens/`는 전체 화면 구성, `src/components/`는 terminal UI 컴포넌트를 담당
모든 인터페이스는 같은 agent loop로 연결되며 UI만 다르고 다 똑같은 소스코드

### 2. Core layer — `query.ts`

핵심 agent loop는 `query.ts`의 `queryLoop()`에서 구현됨
모델 호출 전마다 budget reduction, snip, microcompact 등 5단계 context compaction을 수행 = context engineering
즉 Claude Code의 orchestration과 context 관리를 하는 레이어

### 3. Safety/action layer — `permissions.ts`, `types/hooks.ts`, `tools.ts`, `shouldUseSandbox.ts`

권한 시스템은 `permissions.ts`에서 deny-first 정책으로 동작
`types/hooks.ts`는 hook interception, `tools.ts`는 tool pool 조립, `shouldUseSandbox.ts`는 shell sandbox를 담당
subagent도 일반 tool처럼 실행되며 독립적인 independent context를 사용

### 4. State layer — `context.ts`, `sessionStorage.ts`, `history.ts`, `claudemd.ts`

`context.ts`는 system/user context를 만드는데
`sessionStorage.ts`는 JSONL 세션 저장, `history.ts`는 global history, `claudemd.ts`는 instruction hierarchy를 관리하는 형식
subagent 대화도 별도 sidechain 파일로 저장해 parent context 때문에 inflating, 대화흐름이 오염되는걸 막음

### 5. Backend layer — `BashTool.tsx`, `PowerShellTool.tsx`, `services/mcp/client.ts`

실제 shell 실행, 원격 실행, MCP 연결 등을 담당하는 레이어
MCP는 stdio·HTTP·WebSocket 등 다양한 transport를 지원
`src/tools/` 아래 실제 tool 로직들이 구현됨

---

### 6. QueryEngine — `QueryEngine.ts`

`QueryEngine`은 핵심 엔진이 아니라 비대화형 환경용 conversation wrapper
실제 공통 실행 경로는 `query.ts`의 `query()`와 `queryLoop()`
interactive CLI는 QueryEngine 없이 바로 query()를 호출

---

### 7. 7중 Safety Layer

Claude Code는 7개의 독립 안전 레이어 사용
tool pre-filtering → deny-first rules → permission mode → ML classifier → sandbox → permission reset → hooks 순으로 진행되는데 
하나라도 차단하면 실행되지 않는 defense-in-depth 구조

---

### 8. Context Bottleneck 최적화

Claude Code는 context window를 가장 큰 병목으로 생각하는데
그래서 lazy loading, deferred tool schema, subagent summary-only return 등을 사용
즉 “필요한 정보만 context에 넣는 것”이 핵심 설계 철학이다.

주제: Turn execution에 관해
### 1. Turn Execution / Agentic Loop — `query.ts`

Claude Code는 ReAct 스타일 reactive loop를 사용한다.
`queryLoop()`가 모델 호출 → tool 실행 → 결과 반영을 반복한다.
복잡한 graph search 대신 단순 while-loop 구조로 속도와 단순성을 선택했다.

---

### 2. Query Pipeline — `query.ts`

한 턴은 settings resolution → state init → context assembly → context shaping → model call → tool dispatch → permission check → tool result → stop condition 순서로 진행된다.
`queryLoop()`는 AsyncGenerator 기반이라 streaming UI 출력이 가능하다.
즉 내부는 동기 흐름처럼 유지하면서 외부에는 실시간 스트리밍을 제공한다.

---

### 3. Tool Dispatch — `StreamingToolExecutor.ts`, `toolOrchestration.ts`

tool_use가 나오면 StreamingToolExecutor가 tool을 실시간 병렬 실행한다.
읽기 작업은 병렬 처리하고 shell 같은 state-changing 작업은 직렬 처리한다.
결과 출력 순서는 모델 요청 순서를 그대로 유지한다.

---

### 4. Pre-Model Context Shapers — `query.ts`, `compact.ts`

모델 호출 전 5단계 context shaping이 실행된다.
budget reduction → snip → microcompact → context collapse → auto-compact 순이다.
가벼운 압축부터 시작해 부족할 때만 강한 semantic summary를 수행한다.

---

### 5. Budget Reduction — `applyToolResultBudget()`

너무 긴 tool output을 잘라내고 content reference로 교체한다.
일부 exempt tool은 전체 출력 유지가 가능하다.
resume 시 복구할 수 있도록 replacement도 저장된다.

---

### 6. Snip / Microcompact

Snip은 오래된 history 일부를 제거하는 가벼운 trimming이다.
Microcompact는 시간 기반 + cache-aware 압축을 수행한다.
둘 다 full summary 없이 token 사용량을 줄이는 목적이다.

---

### 7. Context Collapse / Auto-Compact

Context collapse는 전체 history를 유지하면서 모델에게는 축약된 projection만 보여준다.
Auto-compact는 마지막 단계에서 모델이 직접 summary를 생성한다.
즉 “최대한 원본 유지 + 필요할 때만 semantic summarize” 전략이다.

---

### 8. Recovery Mechanisms

출력 token 초과, context overflow, streaming 오류 등에 대한 recovery 로직이 존재한다.
reactive compact와 fallback model도 지원한다.
즉 실패 시 바로 종료하지 않고 자동 복구를 시도한다.

---

### 9. Stop Conditions

loop 종료 조건은:

* tool_use 없음
* maxTurns 도달
* prompt_too_long
* hook 중단
* abort signal

등이다.
즉 모델이 더 이상 action을 요청하지 않으면 턴이 종료된다.
<img width="996" height="326" alt="image" src="https://github.com/user-attachments/assets/bc937c13-fa29-44ed-9f98-1001c9b83d8a" />


주제: Tool Authorization
==============================
### 1. Tool Authorization 철학

Claude Code는 deny-first + layered safety 구조를 사용한다.
사용자는 실제로 permission prompt를 대부분 그냥 승인하기 때문에, 시스템 자체 안전성이 중요하다고 본다.
그래서 권한 규칙·sandbox·classifier를 독립적으로 중첩 적용한다.

---

### 2. Permission Modes — `types/permissions.ts`

7가지 permission mode가 존재한다:

* `plan`
* `default`
* `acceptEdits`
* `auto`
* `dontAsk`
* `bypassPermissions`
* `bubble`

`plan`은 모든 실행 전 승인 필요, `bypassPermissions`는 최소한의 제한만 적용된다.

---

### 3. Deny-first Rule Engine — `permissions.ts`

권한 규칙은 무조건 deny 우선으로 평가된다.
예를 들어 “모든 shell deny”가 있으면 “npm test allow”보다 우선한다.
tool 이름뿐 아니라 Bash(prefix:npm) 같은 content-level matching도 지원한다.

---

### 4. Pre-filtering — `tools.ts`

`filterToolsByDenyRules()`가 금지된 tool을 모델에게 아예 숨긴다.
즉 모델은 forbidden tool 존재 자체를 모른다.
불필요한 tool call 낭비를 막는 목적이다.

---

### 5. Hook System — `types/hooks.ts`

PreToolUse hook은 tool input 수정, deny, ask 등을 수행할 수 있다.
PostToolUse hook은 결과 수정이나 context 추가를 담당한다.
총 27개 hook event 중 5개가 permission flow에 직접 관여한다.

---

### 6. Authorization Pipeline

tool request는:
pre-filtering → hook → deny-first rules → permission handler → classifier → sandbox
순으로 검사된다.
어느 단계 하나라도 차단하면 실행되지 않는다.

---

### 7. Auto-mode Classifier — `yoloClassifier.ts`

ML classifier가 tool invocation 안전성을 자동 평가한다.
conversation transcript와 permission template를 기반으로 allow/deny를 결정한다.
사용자 approval 없이도 자동 차단이 가능하다.

---

### 8. Permission Handler — `useCanUseTool.tsx`

runtime 상황에 따라:

* coordinator
* swarm worker
* speculative classifier
* interactive dialog

중 하나로 분기된다.
interactive mode에서는 사용자 승인창이 기본 fallback다.

---

### 9. Denial as Routing Signal

permission deny는 단순 실패가 아니라 “행동 수정 신호”로 사용된다.
모델은 denial reason을 받고 다음 loop에서 더 안전한 방법을 시도한다.
즉 authorization도 agent behavior shaping 일부다.

---

### 10. Shell Sandboxing — `shouldUseSandbox.ts`

shell command는 별도 sandbox 안에서 실행될 수 있다.
permission approval과 sandboxing은 독립 시스템이다.
허용된 명령도 filesystem/network 제한을 받을 수 있다.

---

### 11. Safety Layer 한계

안전 계층은 독립적이어야 하지만 성능 문제를 공유하기도 한다.
예를 들어 subcommand가 너무 많으면 detailed permission parsing 대신 generic approval로 fallback된다.
즉 defense-in-depth도 성능 제약 때문에 약화될 수 있다.

Extensibility 확장성에 관해
### 1. Extensibility 구조 개요

Claude Code는 확장성을 위해:

* MCP
* Plugins
* Skills
* Hooks

의 4가지 메커니즘을 사용한다.
각각 context cost와 역할이 다르다.

---

### 2. MCP Servers — `services/mcp/config.ts`, `services/mcp/client.ts`

MCP는 외부 서비스/tool 연동의 핵심 메커니즘이다.
stdio, HTTP, WebSocket, SSE 등 다양한 transport를 지원한다.
각 MCP server는 새로운 tool들을 model tool pool에 추가한다.

---

### 3. Plugins — `utils/plugins/schemas.ts`, `pluginLoader.ts`

Plugin은 배포 포맷 + 확장 패키징 시스템 역할을 한다.
commands, agents, skills, hooks, MCP, settings 등 10개 component 타입을 포함할 수 있다.
즉 하나의 plugin이 Claude Code 여러 subsystem을 동시에 확장 가능하다.

---

### 4. Skills — `loadSkillsDir.ts`

Skill은 `SKILL.md` + YAML frontmatter 기반 instruction 패키지다.
allowed tools, model override, execution mode, hooks 등을 정의할 수 있다.
실행 시 SkillTool이 skill instruction을 context에 주입한다.

---

### 5. Hooks — `coreTypes.ts`, `types/hooks.ts`

총 27개 hook event가 존재한다.
permission flow, session lifecycle, context management, subagent coordination 등을 interception 가능하다.
hook은 tool call 차단·수정·retry guidance 제공까지 수행할 수 있다.

---

### 6. Hook Types — `schemas/hooks.ts`

persisted hook은:

* shell command
* LLM prompt
* HTTP
* agent verifier

4종류를 지원한다.
SDK용 callback hook도 별도로 존재한다.

---

### 7. Tool Pool Assembly — `tools.ts`

`assembleToolPool()`이 built-in tool과 MCP tool을 통합한다.
base tools → mode filtering → deny filtering → MCP merge → deduplication 순으로 처리된다.
Claude Code 전체에서 이 함수가 single source of truth 역할을 한다.

---

### 8. Base Tools — `getAllBaseTools()`

최대 54개 tool이 존재한다:

* 19개 항상 활성화
* 35개 조건부 활성화

feature flag, environment, user type에 따라 달라진다.

---

### 9. 왜 4가지 메커니즘인가?

확장 방식마다 context 비용이 다르기 때문이다.
하나의 시스템으로 통합하면 lightweight extension과 heavyweight tool integration을 동시에 효율적으로 처리하기 어렵다.
즉 “표현력 vs context cost” tradeoff 때문에 분리된 구조다.

---

### 10. Context Cost 계층

* Hooks → 거의 0 context
* Skills → 낮음
* Plugins → 중간
* MCP → 높음(tool schema 때문)

즉 값싼 extension은 대규모 사용 가능하고, 비싼 extension은 필요한 경우만 사용한다.

---

### 11. Claude Code의 확장 철학

Claude Code는 “tool만 추가”하는 단순 구조가 아니다.
tool integration, instruction injection, lifecycle interception, packaging을 각각 분리했다.
대신 강력한 확장성을 얻는 대신 학습 난이도는 높아졌다.


주제: Context engineering
=============================
<img width="793" height="464" alt="image" src="https://github.com/user-attachments/assets/ddab80ef-000f-4ab0-b215-351510aaf9e3" />

### 1. Context Construction 철학

Claude Code는 context window를 가장 중요한 희소 자원으로 본다.
그래서 progressive compaction과 file-based memory 구조를 사용한다.
memory도 opaque DB 대신 사용자가 직접 읽고 수정 가능한 파일 기반이다.

---

### 2. Context Window Assembly — `context.ts`, `query.ts`

context는:

* system prompt
* CLAUDE.md
* auto memory
* tool metadata
* conversation history
* tool results

등을 조합해 구성된다.
일부 memory와 MCP instruction은 turn 도중 late injection된다.

---

### 3. System Prompt vs User Context

system prompt는 `appendSystemContext()`로 조립된다.
CLAUDE.md는 system prompt가 아니라 user message로 prepend된다.
즉 CLAUDE.md는 deterministic rule이 아니라 probabilistic guidance 역할이다.

---

### 4. CLAUDE.md Hierarchy — `claudemd.ts`

4단계 memory hierarchy가 존재한다:

1. managed memory
2. user memory
3. project memory
4. local memory

디렉토리 탐색은 current directory에서 root까지 올라가며 수행된다.

---

### 5. Priority / Lazy Loading

directory에 가까운 CLAUDE.md일수록 우선순위가 높다.
nested directory rule은 해당 파일을 읽을 때 lazy loading된다.
즉 코드 탐색에 따라 instruction set도 동적으로 변한다.

---

### 6. File-based Memory 철학

Claude Code memory는 Markdown 파일 기반이다.
사용자가 읽고 수정·삭제·git 관리 가능하다.
vector DB나 embedding retrieval 대신 inspectability를 선택한 구조다.

---

### 7. Auto Memory

embedding retrieval 대신 LLM 기반 memory-file scan을 사용한다.
관련 있는 memory file 최대 5개를 선택한다.
entry 단위가 아니라 file 단위 retrieval이다.

---

### 8. `@include` Directive — `processMemoryFile()`

CLAUDE.md는 `@include`로 modular memory 구성이 가능하다.
relative/home/absolute path 모두 지원한다.
circular reference는 자동 방지된다.

---

### 9. Compaction Pipeline — `query.ts`, `compact.ts`

5단계 compaction이 context overflow를 방지한다:

1. budget reduction
2. snip
3. microcompact
4. context collapse
5. auto-compact

가벼운 압축부터 semantic summary까지 점진적으로 적용한다.

---

### 10. Append-only Transcript 구조

compaction은 기존 transcript를 수정하지 않는다.
boundary marker와 summary를 append-only 방식으로 추가한다.
즉 recovery와 reconstruction이 가능하다.

---

### 11. Compact Recovery Design — `compact.ts`

compact 전 hook이 실행되어 custom instruction 삽입 가능하다.
compact 후에는 runtime state(plan·skills·agents)를 다시 announce한다.
summary로 context를 줄여도 실제 state는 유지된다.

---

### 12. Subagent Delegation — `AgentTool.tsx`

Claude는 복잡한 작업을 subagent에 위임할 수 있다.
subagent는 독립 context window 안에서 실행된다.
즉 parent context를 오염시키지 않으면서 병렬 탐색이 가능하다.

주제: subagent
### 1. Agent Tool — `AgentTool.tsx`

AgentTool은 Claude가 subagent를 생성해 작업을 위임하는 메타-tool이다.
prompt, subagent type, isolation mode, permission override 등을 입력받는다.
SkillTool과 달리 새로운 독립 context window를 생성한다.

---

### 2. Built-in Subagent Types

기본적으로 최대 6개 subagent type이 존재한다:

* Explore
* Plan
* General-purpose
* Claude Code Guide
* Verification
* Statusline-setup

각 agent는 목적별 deny-list와 permission 설정을 가진다.

---

### 3. Custom Agents — `.claude/agents/*.md`, `loadAgentsDir.ts`

사용자는 Markdown 기반 custom agent를 정의할 수 있다.
YAML frontmatter로 tools, model, hooks, permissionMode, skills 등을 설정한다.
즉 agent 하나가 독립적인 mini-Claude subsystem처럼 동작 가능하다.

---

### 4. Skill vs Agent

Skill은 현재 context에 instruction만 추가한다.
Agent는 완전히 새로운 isolated context를 생성한다.
즉 Skill은 lightweight augmentation, Agent는 delegation 구조다.

---

### 5. Isolation Modes — `AgentTool.tsx`

subagent isolation mode는:

* worktree
* remote
* in-process

3종류가 존재한다.
worktree는 Git 기반으로 독립 작업 디렉토리를 만든다.

---

### 6. Worktree Isolation 특징

worktree mode는 Git의 native 기능을 활용한다.
container 없이 filesystem-level isolation을 제공한다.
즉 Docker 없이도 안전한 병렬 코드 수정이 가능하다.

---

### 7. Permission Override — `runAgent.ts`

subagent는 자체 permissionMode를 가질 수 있다.
하지만 parent가 bypassPermissions 같은 강한 mode면 parent 설정이 우선한다.
즉 autonomy escalation은 제한된다.

---

### 8. Async / Background Agents

background agent는 기본적으로 사용자 prompt를 피하려고 한다.
classifier와 hooks가 먼저 자동 판정한다.
필요할 때만 parent terminal에 escalation한다.

---

### 9. Permission Scope 계층

SDK-level allowedTools는 모든 subagent에 유지된다.
하지만 session-level rule은 subagent allowedTools로 교체 가능하다.
즉 global safety + local specialization 구조다.

---

### 10. Sidechain Transcripts — `sessionStorage.ts`, `runAgent.ts`

각 subagent는 별도 `.jsonl` transcript를 가진다.
parent context에는 summary만 반환된다.
즉 debugging은 가능하면서 context explosion은 방지한다.

---

### 11. Summary-only Return 철학

full transcript 공유는 context 폭발 위험이 있다.
그래서 Claude Code는 subagent 결과만 요약해서 parent에 전달한다.
agent team은 일반 세션보다 약 7배 token을 사용하기 때문이다.

---

### 12. Multi-agent Coordination

agent team coordination은 message broker 대신 file locking을 사용한다.
lock-file 기반 mutual exclusion으로 task를 분배한다.
즉 외부 인프라 없이 plain-text JSON만으로 orchestration한다.

openclaw와 비교 
======================
### 1. 비교 대상 구조

Claude Code는 repository 기반 CLI coding agent다.
반면 OpenClaw는 WhatsApp·Slack·Discord 등을 연결하는 persistent gateway system이다.
즉 둘은 “agent runtime 위치” 자체가 다르다.

---

### 2. System Scope 차이

Claude Code는 세션 단위 ephemeral process다.
OpenClaw는 항상 실행되는 WebSocket daemon(control plane)이다.
Claude는 task-centric, OpenClaw는 platform-centric 구조다.

---

### 3. Trust Model 차이

Claude Code는 deny-first permission evaluation을 사용한다.
모델 자체를 잠재적으로 위험한 존재로 본다.
OpenClaw는 trusted operator 기반 perimeter security를 사용한다.

---

### 4. Claude Safety 구조

Claude는:

* per-action permission
* ML classifier
* sandbox
* 7 permission mode

를 조합한다.
trust boundary가 model ↔ execution environment 사이에 있다.

---

### 5. OpenClaw Safety 구조

OpenClaw는:

* DM pairing
* allowlist
* gateway auth

중심이다.
즉 trust boundary가 gateway perimeter에 있다.

---

### 6. Agent Runtime 비교

Claude Code의 중심은 `queryLoop()`다.
모든 인터페이스가 이 loop로 연결된다.
반면 OpenClaw는 gateway 안에 agent runtime이 embedded된다.

---

### 7. Orchestration 차이

Claude는 single-agent 중심 task loop다.
OpenClaw는 multi-channel queue와 RPC dispatch 중심이다.
즉 Claude는 harness, OpenClaw는 orchestration platform에 가깝다.

---

### 8. Extension Architecture 비교

Claude는:

* MCP
* plugins
* skills
* hooks

4계층 확장 구조를 가진다.
context cost 기준으로 설계되었다.

---

### 9. OpenClaw Plugin 구조

OpenClaw는 manifest-first plugin system을 사용한다.
text, speech, media, channels 등 12 capability type을 registry에 등록한다.
즉 gateway 전체 capability를 확장한다.

---

### 10. Memory 철학 비교

둘 다 file-based transparent memory를 사용한다.
Claude는 compaction pipeline에 집중한다.
OpenClaw는 dreaming·daily notes·hybrid retrieval 같은 장기 memory에 집중한다.

---

### 11. Multi-agent 구조 차이

Claude subagent는 parent의 worker 역할이다.
summary만 반환하고 독립 context를 가진다.
OpenClaw는 아예 독립 agent instance들을 routing한다.

---

### 12. Claude Delegation 특징

Claude multi-agent는 task delegation 구조다.
Explore·Plan 같은 subordinate worker를 생성한다.
즉 “하나의 작업을 분할”하는 목적이다.

---

### 13. OpenClaw Multi-agent 특징

OpenClaw는 서로 다른 workspace·channel·user를 담당하는 agent들을 운영한다.
즉 task worker보다 “multi-tenant agent hosting”에 가깝다.
routing 자체가 핵심 기능이다.

---

### 14. Composability

OpenClaw는 Claude Code를 external harness로 호스팅 가능하다.
즉 둘은 경쟁 관계라기보다 layered composition 관계다.
gateway-level system 위에 task-level harness를 올리는 구조다.

---

### 15. 핵심 결론

같은 “AI agent”라도 deployment topology가 다르면:

* safety
* extensibility
* memory
* orchestration

설계가 완전히 달라진다.
Claude는 coding harness 최적화, OpenClaw는 persistent agent platform 최적화다.
