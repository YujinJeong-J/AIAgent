# Context Engineering
이행2본부 AI이행담당 - KTSPACE

## Context Engineering
LLM 기반 에이전트의 성능은 context window에 어떤 정보를 어떻게 담느냐에 따라 크게 달라진다.  
장기 대화나 복잡한 작업에서는 문맥 과부하로 인한 성능 저하, 응답 지연, 오류 발생이 빈번하다.  
이를 방지하고 최적의 성능을 유지하기 위해서는 Context Engineering이 필수적이다.

---

## 멀티 에이전트 메모리 엔지니어링 5대 원칙
1. **Persistence (저장)**: 컨텍스트 외부에 상태 저장, 팀 목표 공유, 절차적 메모리 진화  
2. **Retrieval (선택)**: 임베딩 기반 검색, 에이전트 역할별 맞춤 쿼리, 시급 정보 전파  
3. **Optimization (압축)**: 계층적 요약, 선택적 보존, 캐시 최적화  
4. **Separation (분리)**: 도메인별 메모리 격리, 접근 제어, 세션 경계 설정  
5. **Resolution (동기화)**: 원자적 연산, 버전 관리, 합의 메커니즘, 롤백

---

## 1. Context의 종류
### 1-1) State
현재 대화나 작업의 실시간 상태를 담는 구조체  
- 상황(Context Frame), 활성 툴, 진행 단계, 플래그 값 등 즉시 참조·갱신이 필요한 정보 포함

### 1-2) Short-term memory (Scratchpad)
세션 내 단기 메모 공간  
- 추론 과정, 행동 계획, 중간 결과 등을 저장해 연속 작업에 활용

### 1-3) Long-term Memory
세션 간 유지되는 장기 기억 (User Context 포함)

#### 1-3-1) User Context
- 사용자의 상태, 선호도, 환경 정보 등 개인화된 데이터  
- 예: 프로필 정보, 이력/경험, 선호/습관, 목적/의도, 상황 신호 등

**개인화 적용 방법**  
- 동적 맥락 주입: 대화 시작 시 사용자 요약정보를 LLM에 주입
- 프로필 기반 응답: 톤, 언어, 배경지식 반영
- 이력 기반 추천: 과거 선호·이력 기반 제안
- 차별화된 맥락 비교: 개인 특이성 반영
- 점진적 적응: 대화 중 새 정보로 프로필 갱신

---

## 2. Context 관리 전략
### 2-1) State 관리 전략
- 목표: 실시간 상태를 정확히 추적·제어
- 전략:
  - 상태를 경량 구조체로 유지해 빠른 접근성 확보
    - KV(키-값) + 얕은 JSON/Hash: 깊은 중첩 금지, 1~2 depth 유지
    - 핵심 필드만 보관: current_plan / execution_context / progress / flags / version / updated_at
    - 큰 데이터(툴 결과 원본)는 외부 스토리지로 빼고, 핸들(URI/키)만 상태에 저장
  - 이벤트 기반으로 상태 갱신
    - 이벤트 타입 표준화: TOOL_SUCCESS / TOOL_FAIL / USER_CONFIRM / AUTH_EXPIRED ...
    - Idempotency(멱등성: 같은 작업을 여러 번 반복 실행해도 결과가 한 번 실행했을 때와 동일하게 유지되는 성질): event_id 중복 방지, 이미 처리한 이벤트면 무시
    - 낙관적 락(optimistic lock): 수신 이벤트에 expected_version 포함 → 버전 불일치 시 재시도/병합
  - 불필요 정보 정리로 메모리 최적화
    - 필드 수준 TTL 대신 핸들(참조URL)화 + 세션 TTL + 백그라운드 압축
    - 요약(Summarize) 후 원본 폐기: 대화/툴결과를 요약 문장/통계로 치환
    - 대형 바이너리/JSON은 외부(Blob/S3)에 저장, 상태엔 output_uri만
    -  ex. 툴 원본 결과 10분 보존 → 요약으로 교체
  - 상태 변경 이력 로깅으로 디버깅·회귀 분석 지원
    - 요약 중심의 구조적 로그: 전체 상태 스냅샷 대신 “무엇이 어떻게 바뀌었는가”
        - 읽기/검색이 빠르고 비용이 낮음
        - 회귀 분석 시 바뀐 부분만 순차 재적용하면 상태 재구성이 쉬움 (ex. current_step: 2 → 3)
    - 재현 가능한 최소 이벤트 소싱: 모든 이벤트를 Kafka/NATS에 Append
    - 상관관계 트레이싱: session_id, event_id, actor, prev_version → new_version

### 2-2) Short-term memory 관리 전략
- 목표: reasoning·행동 계획 지원 → LLM이 방금까지의 대화/계산/계획을 저비용/구조적/최신 상태로 유지 가능
- 전략:
  - reasoning과 acting을 위한핵심 구성요소로 활용
  - JSON 구조로 저장해 접근성 확보
  - Conversation Selection → Scene Extraction → CoT Completion → Scene Augmentation 단계 지속 갱신
    - Scene 정보: 현재 대화 주제, 참여자 역할, 환경 설정 값 등
    - Conversation Selection: N개의 대화 기반으로 Scene context 요약
    - Scene Extraction: 요약을 구조화해서 주제/역할/환경을 명확히 함
    - Cot Completion: CoT 과정 짧게 정리
    - Scene Augmentation: 완성된 CoT 중간 결과 바탕으로 다음 액션/계획을 강화하거나 사용자 요청 변화 반영
  - 위 과정을 통해 불필요 토큰 사용 최소화하고 문맥 일관성 유지

### 2-3) Long-term Memory 관리 전략
- 목표: 세션 간 지속되는 지식·경험·사용자 정보 관리
- 핵심 요소: 사용자 프로필, 과거 대화 이력, 선호, 목적, 상황 신호 등.  
- 저장 구분: 활용 목적에 따라 Episodic / Procedural / Semantic 으로 구분해서 최적의 인덱싱과 검색 전략 적용

**Episodic Memory**
- 시간·순서 기반 경험/사건 기록 (시간순, 고유한 맥락)
- 특징: 시간순, 고유한 맥락
- 검색 방식: 시간 범위 + 키워드
- 저장: 타임스탬프 기반 JSON (Elastic Search)  
- 활용: 대화 복원, 과거 이벤트 추천, 회귀 분석

**Procedural Memory**
- 절차·방법·스킬 관련 기억
- 특징: 순서/의존관계 중요
- 검색 방식: 순서/의존관계 탐색
- 저장: 단계별 리스트 + 메타데이터  
- 활용: 자동화 추천, 작업 가이드

**Semantic Knowledge**
- 일반 지식·사실·개념 (시공간 제약 없음)
- 특징: 시공간 제약 없음
- 검색 방식: 의미/유사도 검색
- 저장: 키워드/개념 인덱스 + 벡터 임베딩  
- 활용: RAG 기반 검색, 지식 그래프 확장

---

## 3. Context Persistence (Writing Context)
### 3-3) Long-term Memory 저장 전략
- 위치: Elastic Search  
- 특징: 장기 기억 저장, 대규모 검색 최적화  
- 기능 보완 전략:  
  - 자주 검색되는 데이터 캐싱  
  - 세션 종료된 State 데이터 로그 형태로 장기 저장소에 보존하여 과거 이벤트 추적/회귀 분석에 활용 
  - User Context 생성·갱신 (로그 분석, 병합, 주기 업데이트)
    - state log, scratchpad 기록, long-term memory 과거 대화 이력 분석하여 사용자 행동 패턴/선호도/반복 요청/목적 등을 추출
    - 새 분석 결과와 기존 User Context를 비교/병합 → 충돌 시 최신 데이터 우선 적용
    - 주기적 업데이트
      - 변경된 정보만 증분 업데이트
      - 중요도/빈도 기반 가중치를 적용해 핵심 정보 우선 반영

---

## 4. Context Retrieval (Select Context)
### 4-1) State 활용
- **용도**: 현재 세션의 실시간 상태를 추적해 LLM이 현재 맥락을 정확히 이해하도록 지원
- **접근 방법**: MCP Gateway의 세션 메모리 API를 통해 직접 조회·갱신
- **활용 예시**  
  - 현재 실행 중인 툴과 단계 정보를 프롬프트에 포함  
  - 상태 변화 이벤트를 트리거로 특정 액션 실행

#### JSON 예시
```json
{
  "session_id": "abc123",
  "current_plan": {
    "next_tool": "fetch_market_data",
    "parameters": {
      "date_range": "최근 1개월",
      "symbols": ["AAPL", "GOOG"]
    },
    "expected_output": "market_data_json"
  },
  "execution_context": {
    "is_authenticated": true,
    "has_pending_action": false
  },
  "progress": {
    "current_step": 2,
    "total_steps": 3,
    "step_description": "시장 데이터 수집"
  }
}
```

* **설명**

  * `current_plan`: 다음 호출할 MCP 도구명, 필요한 파라미터, 기대 결과 포함
  * `execution_context`: 인증 상태, 대기 작업 여부 등 실행 조건
  * `progress`: 현재 단계, 전체 단계 수, 단계 설명


### 4-2) Short-term memory (Scratchpad) 활용

* **용도**: 세션 내 reasoning, 행동 계획, 중간 결과를 저장·참조
* **접근 방법**: Redis 키-값 구조를 통해 세션 ID 기반으로 읽기/쓰기
* **활용 예시**

  * 이전 턴의 추론 흐름을 이어받아 다음 응답 생성
  * 중간 계산 결과나 계획을 재활용해 불필요한 재처리 방지

#### 저장 사례

* 다단계 문제 해결을 위한 중간 계산식과 결과값
* 사용자가 요청한 작업 플로우의 단계별 진행 상황
* 대화 중 생성된 임시 변수나 파라미터 값
* Scene 정보(예: 현재 대화 주제, 참여자 역할, 환경 설정 값)
* Chain-of-Thought에서 생성된 논리적 추론 단계

#### JSON 예시

```json
{
  "session_id": "abc123",
  "tool_calls": [
    {
      "tool_name": "fetch_user_profile",
      "reasoning_steps": [
        "사용자 기본 정보 조회",
        "투자 성향 분석"
      ],
      "intermediate_results": {
        "risk_profile": "중간 위험",
        "preferred_assets": ["주식", "채권"]
      },
      "parameters": {
        "user_id": "u001"
      }
    },
    {
      "tool_name": "fetch_market_data",
      "reasoning_steps": [
        "현재 시장 지표 수집",
        "관련 섹터 분석"
      ],
      "intermediate_results": {
        "market_trend": "상승세",
        "sector_focus": ["IT", "자동차"]
      },
      "parameters": {
        "date_range": "최근 1개월"
      }
    },
    {
      "tool_name": "generate_investment_recommendations",
      "reasoning_steps": [
        "사용자 성향과 시장 데이터 결합",
        "추천 상품 리스트 생성"
      ],
      "intermediate_results": {
        "recommendations": [
          {"type": "주식", "name": "삼성전자"},
          {"type": "채권", "name": "현대자동차"}
        ]
      },
      "parameters": {
        "max_results": 5
      }
    }
  ]
}
```

---

### 4-3) Long-term Memory 활용

* **용도**: 장기적 사용자 정보, 과거 대화, 지식 데이터 검색·활용
* **선택/추출 방법**

  * Embedding 기반 유사도 검색으로 현재 질의와 가장 관련성 높은 데이터만 선택
  * Episodic / Procedural / Semantic Memory 유형별로 질의 목적에 맞는 데이터만 추출
  * RAG 기반 파이프라인에서 상위 N개의 고관련성 문서 또는 지식 블록만 반환
* **접근 방법**: Elastic Search 쿼리(Full-text, Vector Search) 또는 RAG 파이프라인
* **활용 예시**

  * 개인화 추천: 사용자 프로필과 과거 대화 이력을 기반으로 맞춤형 제안
  * 이력 검색: 과거 대화나 작업 기록을 검색해 현재 작업에 필요한 정보 제공
  * 지식 제공: Semantic Memory를 활용해 정의, 설명, 관련 지식 그래프 확장
* **운영 방식**

  * 장기 메모리는 독립된 메모리 관리 에이전트 또는 \*\*툴(tool)\*\*로 구현
  * 다른 에이전트가 필요 시 호출하여 데이터 조회·갱신 수행
  * 사용자별 식별자 기반으로 분리 관리하며, 접근 권한을 엄격히 제어
  * 일부 시스템은 비동기 검색을 지원하며, 결과 지연 시 임시 요약 정보로 대체

---

## 5. Context Separation (Isolate Context)
멀티 에이전트 환경에서 Context Separation은 각 에이전트가 불필요한 정보 간섭 없이 자신의 역할에 필요한 맥락만을 처리하도록 보장하는 핵심 전략이다.

* **목표**
  장기 대화나 다중 툴 호출 환경에서 필요한 맥락만을 실시간으로 주입하여 메모리 사용량과 처리 지연을 최소화.

* **구성 요소**

  * **Context Storage**: Redis(In-Memory DB)에 압축된 대화 맥락 저장
  * **MCP Gateway**: 사용자 요청 시 관련 맥락을 검색·압축 해제 후 LLM 입력에 삽입
  * **Context Selector**: 질의와 연관된 Key Block 및 필요한 Plain Block만 선택

* **동작 흐름 (MCP Gateway 기반 동적 Context Pruning)**

  1. MCP Gateway가 질의와 현재 세션 상태를 분석하여 불필요한 맥락을 사전에 제거
  2. Context Storage에서 해당 Context를 로드 및 압축 해제
  3. 압축 해제 시 Selective Compression(keyword filtering) 또는 Block Compression 메타데이터(문단 단위 압축 + 메타데이터 기반 복원) 참조
  4. LLM 입력 프롬프트에 동적으로 삽입하여 응답 생성

* **장점**

  * 불필요한 맥락 제거로 처리 속도 향상
  * 메모리 사용량 절감
  * 맥락 주입의 유연성 확보

---

## 6. Context Compression (Compress Context)

멀티 에이전트 시스템에서 Context Compression은 제한된 컨텍스트 윈도우를
효율적으로 활용하기 위한 필수 기술이다.

### 6-1) 키워드 중요도 기반 필터링

* 빈도수, 사용자 의도, 도구 요구사항 적합도로 중요도 점수 산출
* 점수가 낮은 키워드는 압축 시 배제 또는 통합

### 6-2) 의미론적 클러스터링과 대표 키워드 선정

* 벡터 임베딩으로 의미 유사 키워드를 그룹화
* 그룹 내 대표 키워드만 유지

### 6-3) 시간 기반 요약 및 갱신

* 오래된 키워드 가중치 감소 및 주기적 삭제
* 최신 핵심 키워드를 강조해 컨텍스트 반영 강화

### 6-4) 계층적 압축

* 도메인 → 카테고리 → 세부 키워드 순으로 계층화
* 필요 시 상위 레벨 정보만 유지

### 6-5) JSON 구조화 및 필터링

* 키워드를 JSON 키-값 쌍으로 구조화
* 불필요 세부 키 제거, 필수 키만 포함

#### 압축 사례

**사례 1: 은행 거래 내역 질의**
대화: “지난달 내 계좌 123-456-789 통장에서 출금된 내역 보여줘”

```json
{
  "account_number": "123-456-789",
  "transaction_type": "출금",
  "date_range": "지난달"
}
```

**사례 2: 금융 상품 정보 요청**
대화: “삼성전자 주식과 현대자동차 채권에 투자한 금액 알려줘”
```json
{
  "investments": [
    {"type": "주식", "name": "삼성전자"},
    {"type": "채권", "name": "현대자동차"}
  ],
  "info": "투자금액"
}
```


