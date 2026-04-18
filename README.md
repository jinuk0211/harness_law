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

1. 메모리 시스템
<img width="2434" height="1070" alt="image" src="https://github.com/user-attachments/assets/d70a9d7d-2e36-4b2d-ac25-006ce3966f65" />

Working context. - 현재 작업 - 열린 파일, 중간 변수, 체크포인트
Episodic Experience - 과거 실행 기록 - 실패 원인, 툴 호출 결과, 반성 메모
Semantic Knowledge - 에피소드가 아닌 더 광범위한 지식- 도메인 사실, 프로젝트 컨벤션, RAG corpus
Personalized Memory - 특정 유저/환경에 대한 정보 - 사용자의 선호도, 습관, 반복되는 패턴

3.2 어떻게 외부화하는가 — 아키텍처 진화
4단계 진화 구조야:
① Monolithic Context (초기)

모든 히스토리를 그냥 프롬프트에 때려넣음
단점: 토큰 낭비, 세션 끝나면 증발

② Context + Retrieval Storage (현재 주류)

단기 상태는 컨텍스트에, 장기 기록은 외부 저장소에
RAG가 대표적
문제: 검색 품질이 전부를 좌우함 (잘못 검색하면 오히려 독)
GraphRAG, SYNAPSE 등이 이 검색 품질을 개선하려는 시도

③ Hierarchical Memory (조직화)

모든 기록에 같은 정책 X → 계층적 관리
두 가지 방향:

시공간 분리: 자주 쓰는 건 "핫" 저장소, 나머지는 "콜드" (MemGPT 방식)
기능별 분리: 이벤트/유저 프로필/세계 지식을 별도 채널로 (MemoryBank 방식)



④ Adaptive Memory (최신)

아키텍처 자체가 실행 중에 변함
강화학습으로 검색 전략을 최적화하는 시스템도 등장 (MemRL)


3.3 하네스 시대의 메모리 요구사항
에이전트가 복잡해질수록 메모리에 새로운 요구사항이 생김:

State와 Context 분리 필수: 히스토리를 프롬프트에 다 넣지 말고, 현재 워크스페이스 스냅샷만 읽어야 함 (InfiAgent의 파일 시스템 중심 접근)
스킬 레이어와 통합: 메모리는 증거 보관, 스킬은 그 증거에서 추출된 절차. 스킬 실행 → 새 트레이스 → 다시 메모리에 기록
프로토콜과 연동: 툴 결과, 승인 이벤트 등이 메모리에 정규화되어 저장됨. 메모리 검색 결과가 다음 프로토콜 경로 선택에 영향을 줌
멀티에이전트 거버넌스: 여러 에이전트가 공유 메모리를 쓸 때 읽기/쓰기 권한, 충돌 해결이 필요 → OS 수준의 관리

<img width="2474" height="1226" alt="image" src="https://github.com/user-attachments/assets/3ea65eca-94ec-4b32-8db0-a7a51f06b354" />

# 4장 요약: 스킬 외부화 (Skill Externalization)

---

## 4.1 스킬의 3가지 구성 요소

| 구성 요소 | 역할 |
|-----------|------|
| **Operational Procedure** | 작업 골격 — 단계, 순서, 종료 조건 |
| **Decision Heuristics** | 분기점에서의 판단 규칙 — "뭘 먼저 시도할지" |
| **Normative Constraints** | 수용 가능한 실행 경계 — 안전/컴플라이언스 조건 |

> **툴** = 무엇을 할 수 있는가  
> **스킬** = 어떻게 반복적으로 수행하는가

---

## 4.2 스킬 시스템의 진화 3단계

1. **Stage 1 — Atomic Primitives**: 툴 호출 자체를 안정화 (Toolformer)
2. **Stage 2 — Large-scale Selection**: 많은 툴 중 적절한 것 선택 (Gorilla, ToolLLM)
3. **Stage 3 — Packaged Expertise**: 절차 자체를 재사용 가능한 단위로 패키징 ← **현재**

---

## 4.3 스킬 외부화 프로세스

```
Specification → Discovery → Progressive Disclosure → Execution Binding → Composition
```

- **Specification**: SKILL.md 형태로 능력 경계, 전제조건, 예시 명시
- **Discovery**: 레지스트리에서 의미적으로 적합한 스킬 검색
- **Progressive Disclosure**: 이름 → 요약 → 전체 가이드 순으로 단계적 로딩
- **Execution Binding**: 스킬을 실제 툴/API/서브에이전트에 연결
- **Composition**: 스킬들을 조합해 더 복잡한 작업 처리

---

## 4.4 스킬 획득 방법

| 방법 | 설명 |
|------|------|
| **Authored** | 사람이 직접 작성 (SKILL.md, SOP) |
| **Distilled** | 성공한 실행 궤적에서 패턴 추출 |
| **Discovered** | 환경 탐색 중 자율적으로 발견 (Voyager) |
| **Composed** | 기존 스킬 조합 → 새로운 상위 스킬 생성 |

---

## 4.5 스킬의 한계 (Boundary Conditions)

- **Semantic Alignment**: 스킬 설명과 실제 실행 의도 불일치 가능
- **Portability & Staleness**: API/환경 변경 시 스킬이 무효화됨
- **Unsafe Composition**: 개별적으로 안전한 스킬도 조합 시 보안 취약점 발생 가능
- **Context-dependent Degradation**: 긴 세션에서 스킬 가이드가 노이즈로 작용

---

## 4.6 하네스 내 스킬 동작

```
메모리 → 스킬 선택/파라미터화
스킬 → 프로토콜을 통해 실행 바인딩
실행 결과 → 메모리에 피드백 → 스킬 진화
```

- **메모리**: 스킬 선택의 맥락 제공
- **프로토콜**: 스킬을 실제 액션으로 연결
- **거버넌스**: 권한 체크, 감사 로그, 롤백
- **피드백 루프**: 실행 결과 → 스킬 개선

---

## 핵심 한 줄 요약

> 스킬은 단순한 프롬프트나 툴 래퍼가 아니라, **절차적 전문성을 검사 가능하고 재사용 가능한 외부 구조로 전환하는 인지 아티팩트**다.
