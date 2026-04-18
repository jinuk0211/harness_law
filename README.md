# harness_law
하네스 엔지니어링 관련 논문리뷰 - 10편
Externalization in LLM Agents: A Unified Review of Memory, Skills, Protocols and Harness Engineering
에이젼트의 weight이 아닌 externalization 측면에서의 분석
<img width="996" height="830" alt="image" src="https://github.com/user-attachments/assets/e0b9a3ec-7cb2-427a-8abf-1685de6c0add" />
외재화(externalization)—
인지적 부담을 모델의 내부 연산에서 persistent, inspectable, and reusable한 외부 구조로 점진적으로 이전하는 것—
가 메모리, 스킬, 프로토콜, 하네스 엔지니어링에 걸친 언어 에이전트의 최근 발전들을 통합하는 전환 논리(transition logic)—
각 아키텍처적 전환이 왜 발생했는지, 그리고 어떤 형태의 신뢰성을 보존하려 했는지를 설명하는 메커니즘—
이는 신뢰할 수 있는 에이전시가 어디서 비롯되는가에 관한 주장이다: 
더 거대한 모델만으로부터가 아니라, 내부 역량과 외부 인프라가 요구되는 역량의 전 범위를 함께 충당할 수 있도록 과제 요구를 체계적으로 재구조화하는 것으로부터

1. 메모리 시스템 — 상태(State)의 외부화
문제: LLM은 대화가 끝나면 다 잊어버림
해결: 사용자 선호, 이전 대화 이력, 도메인 지식 등을 외부 저장소에 저장하고 필요할 때 검색
핵심 변환: 재생성 → 검색 (RAG가 대표적 예시)

2. 스킬 시스템 — 절차적 지식의 외부화
문제: 매번 LLM이 "어떻게 할지"를 처음부터 즉흥적으로 생성함
해결: 검증된 절차, 베스트 프랙티스를 **재사용 가능한 컴포넌트(스킬)**로 패키징
핵심 변환: 생성 → 조합
참고로 지금 Claude Skills API(SKILL.md 방식)가 바로 이 개념이야

3. 프로토콜 — 상호작용 구조의 외부화
문제: 에이전트-툴, 에이전트-에이전트 간 통신이 임기응변식으로 되어 있음
해결: 명확한 **머신-리더블 계약(프로토콜)**으로 표준화
핵심 변환: 임기응변 → 구조화된 상호운용
MCP(Model Context Protocol)가 대표적 예시


4. 하네스(Harness) — 통합 런타임 환경
위 세 가지를 묶어주는 엔지니어링 레이어야. 오케스트레이션, 관찰 가능성(observability), 피드백 루프 등을 담당해. 셋이 돌아가는 실행 환경

<img width="1990" height="1100" alt="image" src="https://github.com/user-attachments/assets/18b79ef3-0f67-4cc1-9ad3-68a5ed08e7fd" />
