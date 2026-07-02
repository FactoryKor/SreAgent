# Azure SRE Agent — 스펙 및 비용 구조 정리

> 기준: Microsoft Learn 공식 문서(2026-07 기준). 최신 요율/모델/리전은 변경될 수 있으므로
> [Azure pricing calculator](https://azure.microsoft.com/pricing/details/sre-agent/)에서 재확인 필요.

---

## 1. 요약

- **단일 에이전트가 수용 가능한 리소스 "최대 개수(AKS N대, PostgreSQL N대)"에 대한 공식 상한 수치는 문서에 명시되어 있지 않다.**
- 에이전트가 커버하는 범위는 **개수 제한이 아니라 RBAC 스코프**(리소스 그룹 단위 또는 구독 단위)로 결정된다.
- 한 에이전트가 **스코프 내 다수의 리소스를 동시에 관리 가능**하며, Microsoft는 워크로드를 하나의 에이전트로 통합하는 것을 권장(always-on 비용 절감).
- 공식 문서 예시에서는 한 에이전트가 **"3개 리소스 그룹에 걸친 251개 리소스(Container Apps 5개, AKS 클러스터 2개 등)"** 를 인식하는 사례가 제시됨 — 이는 예시일 뿐 상한이 아님.

---

## 2. 한 에이전트가 커버할 수 있는 범위(스펙)

### 2.1 커버 방식 — "개수"가 아니라 "스코프"

에이전트의 관리 대상은 생성 시 부여하는 **RBAC 접근 범위**로 정해진다.

| 접근 부여 방식 | 설명 |
|---|---|
| **리소스 그룹 단위** | 선택한 리소스 그룹(복수 선택 가능) 내 모든 리소스에 접근 |
| **구독 단위(Subscription-level)** | 매니지드 아이덴티티에 구독 전체 **Reader** 역할 부여 시, 구독 내 모든 리소스 인식 |

- 즉 **AKS 최대 몇 대 / PostgreSQL 최대 몇 대** 같은 하드 코딩된 리소스 타입별 개수 제한은 공식 문서에 존재하지 않는다.
- 실질적 한계는 부여된 스코프의 크기와, 조사(active flow) 시 소비되는 토큰/AAU 비용, 그리고 컨텍스트 윈도우에 의해 결정된다.
- **"단일 에이전트가 여러 워크로드를 처리할 수 있는가?"** → 공식 FAQ 답변: **"Yes. 하나의 에이전트가 스코프 내 여러 리소스를 모니터링할 수 있으며, 워크로드를 하나로 통합하면 별도 에이전트를 여러 개 두는 것보다 always-on 비용이 절감된다."**

> 참고: AKS 진단이나 PostgreSQL 진단을 할 때, 클러스터/인스턴스 대수 자체에 대한 제품 차원의 정량 상한은 문서화되어 있지 않다.
> 대규모 환경이라면 개수 제한이 아니라 **조사 비용(AAU), 리전/스코프 구성, 응답 성능** 관점에서 에이전트를 분리·설계하는 것이 현실적 고려사항이다.

### 2.2 관리 대상 Azure 서비스

| 분류 | 지원 서비스 |
|---|---|
| **컴퓨트** | Virtual Machines, App Service, Container Apps, **AKS**, Azure Functions 등 |
| **스토리지** | Blob storage, File shares, Managed disks, Storage accounts |
| **네트워킹** | Virtual networks, Load balancers, Application gateways, NSG |
| **데이터베이스** | Azure SQL Database, Cosmos DB, **PostgreSQL**, MySQL, Redis |
| **모니터링/관리** | Azure Monitor, Log Analytics, Application Insights, Resource Manager |

- 사실상 모든 **Azure CLI 작업**을 runbook / subagent / agent hook을 통해 자동화 가능.

### 2.3 확장 프리미티브(에이전트 능력 구성 단위)

| 프리미티브 | 설명 |
|---|---|
| **Skills** | 마켓플레이스 runbook, Azure CLI 스크립트 등 개별 기능 |
| **Subagents** | 기본 5종 내장(architecture, logs/metrics, source code, root cause analysis, scanning) + 커스텀 구성 가능 |
| **Python tools** | 커스텀 로직 / 데이터 변환 / API 연동 |
| **MCP servers** | 40+ 커넥터(Datadog, Prometheus, Grafana, New Relic, Splunk, Dynatrace, AWS CloudWatch, GCP 등) 또는 커스텀 |
| **Agent hooks** | 라이프사이클 이벤트 기반 자동화(command hook / prompt hook) |
| **Permission gate** | 모든 툴 호출 전 사전 승인·정책 검증 계층 |

### 2.4 통합(Integrations)

- **모니터링/관측성**: Azure Monitor, Application Insights, Log Analytics, Grafana
- **인시던트 관리**: Azure Monitor Alerts, PagerDuty, ServiceNow
- **소스/CI-CD**: GitHub, Azure DevOps
- **데이터 소스**: Azure Data Explorer(Kusto), MCP 서버
- **커뮤니케이션**: Slack, Microsoft Teams

### 2.5 실행 격리 및 보안(핵심 한계)

- **에이전트당 전용 샌드박스 그룹**(Azure Dedicated Compute 기반 micro VM) — 추론 루프와 툴 실행 분리.
- **고객별 완전 격리**: 컴퓨트(전용 ADC), DB(고객별 Cosmos DB), Blob(에이전트별 스토리지 계정), 네트워크(에이전트별 프록시), 자격증명(에이전트별 매니지드 아이덴티티).
- **코드 실행 환경**: 700+ Python 패키지 사전 설치, **임의 패키지 설치 불가**, egress proxy로 알려진 도메인만 접근 허용.
- **자동 실행 불가 — 사람 승인 필수**: Review 모드는 쓰기 작업에 승인 필요, Autonomous 모드는 매니지드 아이덴티티로 직접 실행.

### 2.6 생성 요구사항 / 제약

| 항목 | 내용 |
|---|---|
| **구독** | `Microsoft.App` 리소스 공급자 등록 필요 |
| **권한** | 구독에 **Owner** 또는 **User Access Administrator** 역할 필요(매니지드 아이덴티티에 RBAC 할당용) |
| **리전** | Sweden Central, East US 2, Australia East에서 리소스 생성 가능해야 함 |
| **네트워크** | `*.azuresre.ai` 방화벽 허용 필요 |
| **자동 생성 리소스** | 에이전트 생성 시 Application Insights, Log Analytics workspace, Managed Identity가 함께 생성됨 |

> **언어 지원 관련 참고**
> 공식 문서(overview)는 "채팅 인터페이스는 영어만 지원(English only)"으로 명시하고 있으나,
> **실사용상 한국어 입력·응답이 동작하는 것으로 확인됨.** 다만 공식 지원 언어가 아니므로 품질/일관성은 보장되지 않을 수 있음.

---

## 3. 비용 구조

과금 단위: **AAU(Azure Agent Unit)** — 모든 프리빌트 Azure 에이전트에서 공통으로 쓰이는 표준 처리 측정 단위.
월 청구는 아래 두 흐름의 합으로 구성됨.

### 3.1 Always-on flow (고정 비용)

| 항목 | 요율 |
|---|---|
| Always-on flow | **에이전트당 4 AAU / 시간** |

- 실제 작업 여부와 무관하게 에이전트가 존재하는 한 부과(생성 → **삭제** 시까지).
- **Stop 해도 always-on 비용은 계속 발생** → 완전 중단하려면 **Delete** 필요.

### 3.2 Active flow (변동 비용) — 처리 중 토큰 소비 기반

4가지 토큰 타입을 각각 계량 후 합산: **Input / Output / Cache read / Cache write**

**모델별 AAU 요율 (100만 토큰당)**

| 모델 | Input | Output | Cache read | Cache write |
|---|---|---|---|---|
| Claude Opus 4.6 | 100 | 500 | 10 | 125 |
| GPT 5.3 Codex | 35 | 280 | 3.5 | 0 |
| GPT 5.2 | 35 | 280 | 3.5 | 0 |

- **처리 시간만 과금** — 사용자 응답을 기다리는 시간은 미청구.
- **매월 초 소비 카운터 리셋.**
- 모델 provider는 에이전트 설정에서 지정(선택 모델이 요율 결정).

### 3.3 작업 유형별 소비 예시

| 시나리오 | Input | Output | Claude Opus 4.6 | GPT 5.3 Codex | 예시 |
|---|---|---|---|---|---|
| Quick question | ~20K | ~2K | ~3.8 AAU | ~1.3 AAU | "최근 알림 보여줘" |
| Incident investigation | ~200K | ~15K | ~35.3 AAU | ~11.7 AAU | Azure Monitor 자동 인시던트 |
| Full remediation | ~500K | ~40K | ~86.5 AAU | ~30.1 AAU | "실패한 배포 진단·수정" |

**계산 예시 (Claude Opus 4.6, Quick question)**

| 토큰 타입 | 토큰 | 100만당 요율 | AAU |
|---|---|---|---|
| Input | 20K | 100 | 2.0 |
| Output | 2K | 500 | 1.0 |
| Cache read | 15K | 10 | 0.15 |
| Cache write | 5K | 125 | 0.625 |
| **합계** | | | **3.775 AAU** |

### 3.4 비용 관리 / 모니터링

- **Active flow 지출 한도**: 월간 **500 ~ 1,000,000 AAU** 범위 설정. 한도 도달 시 해당 월 채팅·액션 중단(always-on은 계속). 증액은 즉시, 감액은 다음 달 적용.
- **모니터링**: Settings > Agent consumption(스레드별/일별 소비), Azure Cost Management.

**액션별 과금 영향**

| 액션 | Active flow | Always-on | 다음 달 재개 방법 |
|---|---|---|---|
| 예산 한도 도달 | 중단 | 계속 청구 | 월 초 자동 리셋 |
| Stop 에이전트 | 중단 | 계속 청구 | Settings > Basics에서 Start |
| Delete 에이전트 | 중단 | 중단 | 신규 에이전트 생성 |

### 3.5 비용 최적화 팁

- 컨텍스트(skills/knowledge/문서) 추가로 토큰 낭비 감소.
- Response plan으로 인시던트를 심각도·서비스·키워드 기준 필터링.
- 스케줄 작업으로 배치화(상시 폴링 대신 일/주 단위).
- 자동화 전 chat/Playground에서 프롬프트 검증.
- 유휴 에이전트는 Stop, 미사용 에이전트는 Delete.

### 3.6 무료 티어

- **무료 티어 없음.** 에이전트 **생성 시점부터 과금 시작.**
- 지역별 가격 차이 가능.

---

## 4. 결론 — "최대 몇 대까지?"에 대한 답

- Microsoft 공식 문서 기준 **리소스 타입별(AKS, PostgreSQL 등) 개수 상한은 규정되어 있지 않음.**
- 하나의 에이전트로 **구독 전체 또는 여러 리소스 그룹을 스코프로 지정하여 다수 리소스를 동시 관리 가능.**
- 실무적 한계는 다음 요인으로 나타남:
  1. **비용(AAU)** — 진단 대상/빈도가 늘수록 active flow AAU 증가.
  2. **컨텍스트 윈도우** — 한 번의 조사에서 다룰 수 있는 데이터량 한계.
  3. **리전/스코프 구성** — 에이전트는 특정 리전에 생성되며 RBAC로 범위 제한.
- 따라서 대규모 환경에서는 "몇 대까지 되는가"보다 **비용·성능·관리 경계 기준으로 에이전트를 어떻게 분할/통합할지**를 설계하는 것이 핵심.

---

## 참고 출처 (Microsoft Learn)

- Overview: https://learn.microsoft.com/en-us/azure/sre-agent/overview
- Pricing and billing: https://learn.microsoft.com/en-us/azure/sre-agent/pricing-billing
- Security overview: https://learn.microsoft.com/en-us/azure/sre-agent/security-overview
- Create agent: https://learn.microsoft.com/en-us/azure/sre-agent/create-agent
- Pricing calculator: https://azure.microsoft.com/pricing/details/sre-agent/
