# Chapter 4. 관측 가능성 한 번에 구축하기

> 책: AI 시대에 개발자가 알아야 할 인프라 구성 배포 with 클로드코드  
> 읽은 범위: Chapter 4, 관측 가능성 한 번에 구축하기

## 한 줄 요약

관측 가능성은 메트릭, 로그, 트레이스, 프로파일을 통해 시스템 상태와 문제 원인을 파악하는 기반이며, Kubernetes 환경에서는 Prometheus, Grafana, Loki, Alertmanager 등을 조합해 운영 가능한 관측 체계를 만들 수 있다.

---

## 이 장에서 다룬 내용

Chapter 4에서는 Kubernetes 환경에서 관측 가능성을 한 번에 구축하는 흐름을 다룬다.

단순히 모니터링 도구를 설치하는 것이 아니라, 애플리케이션과 클러스터의 상태를 지속적으로 수집하고, 문제가 발생했을 때 원인을 찾고, 필요한 경우 알림까지 받을 수 있는 기반을 만드는 과정이다.

핵심적으로 다룬 내용은 다음과 같다.

- 관측 가능성의 세 가지 요소
  - 메트릭
  - 로그
  - 트레이스
- 네 번째 요소로 볼 수 있는 프로파일
- Prometheus와 Grafana를 활용한 Kubernetes 모니터링
- `kube-prometheus-stack` Helm 차트
- Pull 기반 메트릭 수집
- PromQL
- ServiceMonitor
- kube-state-metrics
- Prometheus 장기 저장 전략
- Thanos, Grafana Mimir, VictoriaMetrics 비교
- Loki와 Fluent Bit를 활용한 중앙 로그 수집
- LogQL
- PrometheusRule과 Alertmanager를 활용한 알림

---

## 기대했던 장

개인적으로 이번 장은 가장 기대했던 챕터 중 하나였다.

그 이유는 “관측 가능성을 자연어로 어떻게 구축할 수 있을까?”라는 궁금증도 있었지만, 그보다 먼저 **관측 가능성 자체를 어떻게 제대로 구축할 것인가**에 대한 고민이 있었기 때문이다.

운영 환경에서 중요한 것은 단순히 애플리케이션을 배포하는 것이 아니다.

배포한 애플리케이션이 다음 상태를 계속 확인할 수 있어야 한다.

```text
정상적으로 동작하고 있는가?
느려지고 있지는 않은가?
에러가 증가하고 있지는 않은가?
리소스를 과도하게 사용하고 있지는 않은가?
문제가 생겼을 때 원인을 추적할 수 있는가?
필요한 사람에게 제때 알림을 보낼 수 있는가?
```

따라서 메트릭 수집, 로그 통합, 알림 설정까지 Claude Code를 통해 한 번에 구축한다는 흐름이 흥미롭게 느껴졌다.

---

## 관측 가능성이란 무엇인가

관측 가능성, 즉 Observability는 시스템 내부 상태를 외부에서 수집한 데이터로 이해할 수 있게 만드는 능력이다.

일반적으로 관측 가능성은 세 가지 요소로 설명된다.

```text
메트릭
로그
트레이스
```

최근에는 여기에 프로파일을 추가해 네 번째 요소로 보기도 한다.

---

## 메트릭

메트릭은 숫자 데이터다.

예를 들면 다음과 같다.

```text
CPU 사용률 90%
초당 요청 수 500개
에러율 2%
메모리 사용량 75%
응답 시간 p95 300ms
```

메트릭은 시스템이 지금 정상인지 비정상인지 판단하는 데 사용된다.

특히 다음과 같은 질문에 답할 수 있다.

```text
현재 트래픽은 얼마나 들어오고 있는가?
CPU나 메모리는 얼마나 사용 중인가?
에러율이 평소보다 증가했는가?
응답 시간이 느려졌는가?
Pod 수가 원하는 개수만큼 유지되고 있는가?
```

메트릭은 숫자로 표현되기 때문에 대시보드, 알림, 추세 분석에 적합하다.

운영자는 메트릭을 통해 시스템의 현재 상태를 빠르게 파악할 수 있다.

---

## 로그

로그는 텍스트 데이터다.

예를 들면 다음과 같다.

```text
Connection refused
timeout after 30s
failed to connect database
invalid token
Exception at line 42
```

메트릭이 “문제가 발생했다”는 신호를 준다면, 로그는 “무엇이 잘못되었는가”를 파악하는 데 도움을 준다.

예를 들어 에러율이 갑자기 증가했다는 메트릭을 확인했다면, 다음 단계에서는 로그를 확인해 구체적인 원인을 찾아야 한다.

```text
어떤 요청에서 에러가 발생했는가?
어떤 예외가 발생했는가?
외부 API 호출이 실패했는가?
DB 연결이 끊겼는가?
인증이나 권한 문제가 있었는가?
```

로그는 원인 분석에 강하지만, 양이 많아질수록 검색과 필터링 체계가 중요해진다.

---

## 트레이스

트레이스는 요청의 흐름을 추적하는 데이터다.

하나의 요청이 여러 서비스를 거쳐 처리될 때, 각 서비스에서 얼마나 시간이 걸렸는지 확인할 수 있다.

예를 들어 사용자가 결제 요청을 보냈다고 가정하면 요청은 다음과 같은 서비스를 거칠 수 있다.

```text
API Gateway
→ Order Service
→ Payment Service
→ Notification Service
→ External Payment API
```

트레이스는 이 요청이 어떤 경로를 거쳤고, 어느 구간에서 느려졌는지를 보여준다.

메트릭이 전체적인 상태를 보여주고, 로그가 개별 이벤트의 원인을 보여준다면, 트레이스는 분산 시스템에서 요청의 흐름을 보여준다.

특히 마이크로서비스 환경에서는 트레이스가 매우 중요하다.

서비스가 많아질수록 하나의 장애가 어디서 시작되었는지 파악하기 어려워지기 때문이다.

---

## 프로파일

관측 가능성의 네 번째 요소로 프로파일을 꼽기도 한다.

프로파일은 CPU나 메모리를 어떤 함수가 얼마나 사용하고 있는지 코드 수준에서 보여준다.

예를 들어 다음과 같은 질문에 답할 수 있다.

```text
어떤 함수가 CPU를 많이 사용하고 있는가?
어떤 코드 경로에서 메모리 사용량이 증가하는가?
특정 요청을 처리할 때 병목이 어디에서 발생하는가?
```

메트릭, 로그, 트레이스가 시스템과 요청 흐름을 관찰하는 데 초점이 있다면, 프로파일은 더 깊이 들어가 코드 수준의 성능 병목을 찾는 데 유용하다.

대표적인 도구로는 Pyroscope 같은 도구가 있다.

---

## 관측 가능성 요소 정리

관측 가능성의 각 요소를 정리하면 다음과 같다.

| 요소 | 데이터 형태 | 주요 목적 | 예시 |
|---|---|---|---|
| 메트릭 | 숫자 | 현재 상태 판단 | CPU 사용률, 요청 수, 에러율 |
| 로그 | 텍스트 | 문제 원인 분석 | 에러 메시지, 예외, 이벤트 기록 |
| 트레이스 | 요청 흐름 | 병목 구간 추적 | 서비스 간 호출 경로와 지연 시간 |
| 프로파일 | 코드 수준 사용량 | 성능 병목 분석 | 함수별 CPU/메모리 사용량 |

각 요소는 서로 대체 관계가 아니라 보완 관계다.

문제가 발생했을 때의 흐름은 보통 다음과 같다.

```text
메트릭으로 이상 징후 감지
→ 로그로 원인 확인
→ 트레이스로 느린 구간 추적
→ 프로파일로 코드 수준 병목 분석
```

---

## Prometheus와 Grafana

Kubernetes 환경에서 메트릭 수집과 시각화를 구성할 때 Prometheus와 Grafana 조합은 사실상 표준처럼 사용된다.

Prometheus는 메트릭을 수집하고 저장하는 역할을 한다.  
Grafana는 수집된 메트릭을 대시보드로 시각화하는 역할을 한다.

```text
Prometheus = 메트릭 수집과 저장
Grafana    = 메트릭 시각화
```

Kubernetes 환경에서는 `kube-prometheus-stack` Helm 차트를 사용해 관련 구성 요소를 한 번에 설치할 수 있다.

이 차트에는 보통 다음과 같은 구성 요소들이 포함된다.

- Prometheus
- Grafana
- Alertmanager
- Prometheus Operator
- node-exporter
- kube-state-metrics

각 구성 요소는 역할이 다르다.

| 구성 요소 | 역할 |
|---|---|
| Prometheus | 메트릭 수집과 저장 |
| Grafana | 대시보드 시각화 |
| Alertmanager | 알림 라우팅과 관리 |
| Prometheus Operator | Prometheus 관련 리소스 관리 |
| node-exporter | 노드 수준 메트릭 수집 |
| kube-state-metrics | Kubernetes 오브젝트 상태를 메트릭으로 변환 |

---

## Prometheus와 Grafana를 선택하는 이유

Prometheus와 Grafana 조합이 많이 사용되는 이유는 다음과 같이 정리할 수 있다.

| 이유 | 설명 |
|---|---|
| Kubernetes 생태계 표준 | Kubernetes와 함께 많이 사용되며 생태계가 크다. |
| 자체 호스팅 가능 | SaaS 구독 없이 직접 운영할 수 있다. |
| Helm 기반 설치 | `kube-prometheus-stack`으로 검증된 구성 요소를 한 번에 설치할 수 있다. |
| Grafana 통합 | 대시보드 시각화가 쉽고, Loki나 Tempo 같은 도구와도 연결할 수 있다. |
| 확장성 | Alertmanager, Thanos, Mimir 등과 연계해 확장할 수 있다. |

특히 자체 호스팅이 가능하다는 점은 중요한 장점이다.

외부 SaaS 사용이 어렵거나 내부망 중심으로 운영해야 하는 환경에서는 Prometheus와 Grafana를 직접 운영하는 방식이 더 적합할 수 있다.

---

## Pull 기반 수집

Prometheus의 중요한 특징 중 하나는 Pull 기반 수집 방식이다.

일반적으로 애플리케이션이나 exporter는 `/metrics` 엔드포인트를 노출한다.  
Prometheus는 주기적으로 이 엔드포인트를 호출해 메트릭을 가져온다.

```text
Prometheus
→ 각 Pod 또는 Service의 /metrics 호출
→ 메트릭 수집
→ 시계열 데이터로 저장
```

즉, 대상이 Prometheus로 데이터를 보내는 것이 아니라, Prometheus가 대상을 찾아가서 긁어오는 방식이다.

이 과정을 scrape이라고 부른다.

예를 들면 다음과 같은 흐름이다.

```text
애플리케이션 Pod가 /metrics 노출
→ Prometheus가 30초마다 /metrics scrape
→ 수집된 메트릭을 Prometheus TSDB에 저장
→ Grafana에서 PromQL로 조회
```

이 방식은 `kubectl top`과는 방향이 다르다.

`kubectl top`은 현재 리소스 사용량을 조회하는 명령에 가깝지만, Prometheus는 지속적으로 메트릭을 수집하고 저장한다.

---

## PromQL

PromQL은 Prometheus Query Language의 약자다.

Prometheus에 저장된 시계열 데이터를 조회하고 집계하기 위한 쿼리 언어다.

예를 들어 HTTP 요청 수를 5분 단위 증가율로 보고 싶다면 다음과 같은 쿼리를 사용할 수 있다.

```promql
rate(http_requests_total[5m])
```

에러율을 계산하는 쿼리는 다음과 비슷한 형태가 될 수 있다.

```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

PromQL을 잘 다룰 수 있으면 단순히 메트릭을 보는 것을 넘어, 서비스 상태를 더 의미 있게 해석할 수 있다.

예를 들어 다음과 같은 질문에 답할 수 있다.

```text
최근 5분 동안 에러율은 얼마인가?
특정 API의 p95 응답 시간은 얼마인가?
Pod별 CPU 사용량은 어떻게 다른가?
배포 이후 요청 수나 에러율이 바뀌었는가?
```

---

## ServiceMonitor

`ServiceMonitor`는 어떤 Service를 Prometheus가 scrape할지 선언하는 Custom Resource이다.

일반적으로 Prometheus Operator와 함께 사용된다.

기존 Prometheus 설정에서는 scrape 대상을 직접 설정 파일에 작성해야 한다.  
하지만 Kubernetes 환경에서는 ServiceMonitor를 사용해 scrape 대상을 Kubernetes 리소스로 선언할 수 있다.

예를 들어 애플리케이션이 `/metrics` 엔드포인트를 노출하고 있다면, ServiceMonitor를 통해 Prometheus에게 다음과 같이 알려줄 수 있다.

```text
이 namespace의
이 label을 가진 Service를
이 path와 port로 scrape해라.
```

이 방식은 GitOps와도 잘 어울린다.

ServiceMonitor 역시 Kubernetes Manifest로 작성할 수 있기 때문에, Git에 선언하고 리뷰한 뒤 배포할 수 있다.

---

## kube-state-metrics

`kube-state-metrics`는 Kubernetes 오브젝트의 상태를 메트릭으로 변환하는 도구다.

예를 들어 다음과 같은 Kubernetes 상태 정보를 Prometheus 메트릭으로 노출한다.

- Pod 개수
- Deployment replica 수
- Desired replica 수
- Ready 상태의 Pod 수
- Node 상태
- Job 상태
- PersistentVolumeClaim 상태

node-exporter가 노드의 CPU, 메모리, 디스크 같은 시스템 메트릭을 제공한다면, kube-state-metrics는 Kubernetes 리소스의 선언 상태와 현재 상태를 메트릭으로 제공한다.

예를 들어 다음과 같은 질문에 답할 수 있다.

```text
원하는 replica 수와 실제 준비된 replica 수가 같은가?
특정 Deployment의 Pod가 몇 개 Ready 상태인가?
Pending 상태의 Pod가 있는가?
Job이 실패했는가?
PVC가 Bound 상태인가?
```

Kubernetes 운영에서는 리소스 사용량뿐만 아니라 오브젝트의 상태도 중요하기 때문에, kube-state-metrics는 필수적인 구성 요소에 가깝다.

---

## Prometheus 장기 저장에 대한 고민

Prometheus는 기본적으로 로컬 디스크에 시계열 데이터를 저장한다.

짧은 기간의 메트릭을 보관하고 조회하는 데는 적합하지만, 프로덕션 환경에서는 더 긴 기간의 메트릭을 보관해야 하는 경우가 많다.

예를 들면 다음과 같은 요구사항이 있을 수 있다.

```text
최근 1년간 트래픽 추이를 보고 싶다.
장애 발생 전후의 메트릭을 장기간 보관하고 싶다.
월별 리소스 사용량을 비교하고 싶다.
감사나 리포팅을 위해 일정 기간 데이터를 보존해야 한다.
```

이런 경우 Prometheus 단독 구성만으로는 한계가 있다.

Prometheus의 로컬 저장소를 무한히 키우는 방식은 운영 부담이 커질 수 있기 때문이다.

따라서 장기 저장을 위해 별도의 저장 계층이나 확장 도구를 함께 고려해야 한다.

---

## 폐쇄망 또는 온프레미스 환경에서의 장기 저장

운영 환경에 따라 외부 SaaS 기반 모니터링 서비스를 사용하기 어려운 경우가 있다.

예를 들어 다음과 같은 조건이 있을 수 있다.

```text
외부 네트워크 접근이 제한된다.
자체 호스팅 오픈소스 중심으로 구성해야 한다.
S3 호환 오브젝트 스토리지가 이미 있다.
내부 컨테이너 레지스트리나 아티팩트 저장소를 사용한다.
```

이런 조건에서는 “Prometheus 데이터를 장기 저장하려면 어떤 구조가 적합한가?”를 고민하게 된다.

핵심은 이미 가지고 있는 자산을 어떻게 활용할 것인지다.

| 보유 자산 | 역할 | 의미 |
|---|---|---|
| S3 호환 오브젝트 스토리지 | 장기 저장 백엔드 | 메트릭 블록을 장기간 보관할 수 있음 |
| 내부 컨테이너 레지스트리 | 이미지 반입과 배포 | 외부 이미지를 내부 환경으로 가져와 사용할 수 있음 |
| Helm 차트 저장소 | 차트 반입과 배포 | 외부 Helm 차트를 내부 저장소로 관리 가능 |

이 조건에서는 S3를 장기 저장소로 적극 활용할 수 있는 엔진을 선택하는 것이 중요하다.

---

## Prometheus 장기 저장 후보 비교

Prometheus 장기 저장을 위해 자주 검토되는 선택지는 다음과 같다.

- Thanos
- Grafana Mimir
- VictoriaMetrics

각 도구는 메트릭을 저장하고 조회하는 방식이 조금씩 다르다.

| 엔진 | S3 사용 방식 | 적합한 상황 |
|---|---|---|
| Thanos | Prometheus 블록을 S3에 업로드하고 장기 조회 계층으로 사용 | 기존 Prometheus 유지 + S3 장기 저장 |
| Grafana Mimir | 오브젝트 스토리지 중심의 대규모 메트릭 저장소 | 초대규모, 멀티테넌트 환경 |
| VictoriaMetrics | 로컬 디스크 중심 저장, S3는 주로 백업 용도로 활용 | 단순 운영, 적은 구성 요소, 로컬 디스크 중심 |

이미 Prometheus를 사용하고 있고 S3 호환 오브젝트 스토리지가 있다면, Thanos가 자연스러운 선택지가 될 수 있다.

이유는 기존 Prometheus 구조를 크게 바꾸지 않고, Sidecar를 붙여 장기 저장을 확장할 수 있기 때문이다.

---

## Thanos를 이용한 장기 저장 구조

Thanos는 기존 Prometheus에 장기 저장과 글로벌 조회 기능을 추가하는 방식으로 사용할 수 있다.

기본 구조는 다음과 같다.

```text
Prometheus
  |
  | scrape
  v
최근 메트릭 로컬 저장
  |
  | Thanos Sidecar
  v
S3 호환 오브젝트 스토리지
  |
  | Thanos Store Gateway
  v
과거 데이터 조회

Thanos Query
  |
  | 최근 데이터 + 과거 데이터 통합 조회
  v
Grafana
```

조금 더 자세히 보면 다음과 같다.

```text
Prometheus
  ├─ 최근 데이터 로컬 저장
  └─ Thanos Sidecar
       └─ TSDB 블록을 S3에 업로드

S3 호환 오브젝트 스토리지
  ├─ 장기 메트릭 블록 저장
  └─ 과거 데이터 보관

Thanos Store Gateway
  └─ S3에 저장된 과거 데이터를 쿼리 가능하게 노출

Thanos Compactor
  ├─ 오래된 데이터 다운샘플링
  └─ retention 정책 적용

Thanos Query
  ├─ Prometheus Sidecar의 최근 데이터 조회
  ├─ Store Gateway의 과거 데이터 조회
  └─ 여러 소스를 하나의 뷰로 통합

Grafana
  └─ Thanos Query를 데이터소스로 사용
```

---

## Thanos 컴포넌트 역할

Thanos를 구성하는 주요 컴포넌트는 다음과 같다.

| 컴포넌트 | 역할 |
|---|---|
| Sidecar | Prometheus 옆에 붙어 TSDB 블록을 오브젝트 스토리지로 업로드 |
| Store Gateway | 오브젝트 스토리지에 저장된 과거 데이터를 쿼리 가능하게 노출 |
| Compactor | 오래된 데이터 다운샘플링과 retention 관리 |
| Query | 여러 Prometheus와 Store Gateway를 하나의 조회 계층으로 통합 |
| Query Frontend | 대규모 조회 성능 개선과 캐싱에 사용 가능 |
| Ruler | 장기 저장 데이터 기반의 룰 평가에 사용 가능 |

특히 Sidecar와 Store Gateway가 핵심이다.

Sidecar는 Prometheus와 S3 사이를 연결하고, Store Gateway는 S3에 저장된 데이터를 다시 조회 가능하게 만든다.

Compactor는 장기 저장에서 매우 중요하다.

무작정 데이터를 계속 저장하면 비용과 용량이 증가하기 때문에, 오래된 데이터는 다운샘플링하고 보존 기간을 명확히 정해야 한다.

예를 들면 다음과 같은 정책을 생각해볼 수 있다.

```text
Raw 데이터: 30일 보관
5분 단위 다운샘플 데이터: 90일 보관
1시간 단위 다운샘플 데이터: 1년 보관
```

---

## 장기 저장 선택 기준

세 도구를 선택 기준으로 다시 정리하면 다음과 같다.

| 상황 | 더 적합한 선택 |
|---|---|
| 기존 Prometheus 유지 + S3 장기 저장 부착 | Thanos |
| 초대규모 메트릭 + 멀티테넌트 + 오브젝트 스토리지 중심 | Grafana Mimir |
| 운영 부품 최소화 + 로컬 디스크 중심 + S3는 백업 용도 | VictoriaMetrics |

대부분의 중간 규모 운영 환경에서는 기존 Prometheus를 유지하면서 Thanos를 붙이는 방식이 가장 자연스러운 진화 경로처럼 보인다.

반면 수많은 팀과 대규모 시계열을 처리해야 하는 플랫폼 수준의 환경이라면 Mimir를 검토할 수 있다.

운영 단순성이 가장 중요하고 장기 저장 조회 요구가 크지 않다면 VictoriaMetrics도 좋은 선택지가 될 수 있다.

---

## 내부 레지스트리와 아티팩트 저장소 활용

외부 네트워크 접근이 제한된 환경에서는 이미지와 Helm 차트를 내부로 반입하는 절차도 중요하다.

Prometheus, Grafana, Thanos 같은 도구를 설치하려면 컨테이너 이미지와 Helm 차트가 필요하다.  
인터넷에 직접 접근할 수 없다면 내부 레지스트리나 아티팩트 저장소를 통해 배포해야 한다.

일반적인 흐름은 다음과 같다.

```text
외부 이미지와 Helm 차트 확인
→ 내부 보안 기준에 맞게 반입
→ 내부 레지스트리에 이미지 저장
→ 내부 Helm 저장소에 차트 저장
→ Kubernetes 클러스터는 내부 저장소만 참조
```

내부 레지스트리와 아티팩트 저장소는 다음 역할을 한다.

| 구성 요소 | 역할 |
|---|---|
| 컨테이너 레지스트리 | Prometheus, Grafana, Thanos 등의 이미지 저장 |
| Helm 저장소 | kube-prometheus-stack, Thanos 차트 저장 |
| Raw 저장소 | 오프라인 번들, 바이너리, 설정 파일 보관 |
| Proxy cache | 외부 이미지를 내부에 캐싱하거나 반입 자동화 |

이 구조를 잘 만들어두면 폐쇄망이나 제한된 네트워크 환경에서도 오픈소스 기반 관측 가능성 스택을 안정적으로 운영할 수 있다.

---

## 로그 수집이 필요한 이유

메트릭은 “무엇이 잘못됐는지”를 알려주지만, “왜 잘못됐는지”는 로그에서 찾아야 한다.

Prometheus로 에러율 증가, 응답 시간 증가, Pod 재시작 같은 이상 징후를 확인할 수 있다.  
하지만 실제 원인을 찾으려면 로그를 봐야 한다.

Kubernetes 환경에서 로그를 중앙 수집하지 않으면 다음 문제가 생긴다.

```text
Pod마다 로그를 따로 확인해야 한다.
Pod가 재시작되면 이전 로그를 잃을 수 있다.
여러 서비스의 로그를 한 번에 검색하기 어렵다.
어제 발생한 에러를 찾기 어렵다.
장애 시점의 로그를 보존하기 어렵다.
```

예를 들어 다음 명령으로 Pod 로그를 볼 수는 있다.

```bash
kubectl logs <POD_NAME>
```

하지만 Pod가 많아지고 서비스가 늘어나면 이 방식만으로는 운영이 어렵다.

따라서 로그도 메트릭처럼 중앙에서 수집하고 검색할 수 있어야 한다.

---

## Loki와 Fluent Bit

중앙 로그 수집을 위한 선택지로 Loki와 Fluent Bit 조합을 사용할 수 있다.

역할은 다음과 같이 나눌 수 있다.

```text
각 노드의 Pod 로그
  |
  v
Fluent Bit
  |
  v
Loki
  |
  v
Grafana
```

- Fluent Bit는 각 노드에서 로그를 수집하고 전송한다.
- Loki는 로그를 저장하고 검색할 수 있게 한다.
- Grafana는 Loki를 데이터소스로 연결해 로그를 조회한다.

Fluent Bit는 보통 DaemonSet으로 배포된다.

DaemonSet은 각 노드마다 Pod를 하나씩 실행하는 Kubernetes 리소스다.  
따라서 Fluent Bit를 DaemonSet으로 배포하면 모든 노드에서 컨테이너 로그를 수집할 수 있다.

```text
node-exporter가 각 노드의 시스템 메트릭을 수집한다면,
Fluent Bit는 각 노드의 컨테이너 로그를 수집한다.
```

---

## Loki를 선택하는 이유

Loki는 Grafana Labs에서 만든 로그 저장 시스템이다.

특징은 “로그를 위한 Prometheus”처럼 설계되었다는 점이다.

| 항목 | Prometheus | Loki |
|---|---|---|
| 대상 | 메트릭 | 로그 |
| 인덱싱 | 라벨 | 라벨 |
| 쿼리 언어 | PromQL | LogQL |
| Grafana 통합 | 가능 | 가능 |

Loki는 로그 내용 전체를 인덱싱하지 않는다.

대신 다음과 같은 라벨만 인덱싱한다.

```text
namespace
pod
container
app
job
```

그리고 검색할 때는 라벨로 범위를 좁힌 뒤, 해당 로그 내용에서 텍스트를 필터링한다.

이 방식은 Elasticsearch 기반 로그 시스템보다 상대적으로 가볍게 운영할 수 있다는 장점이 있다.

---

## Loki의 라벨 기반 인덱싱

Loki의 핵심은 라벨 기반 인덱싱이다.

예를 들어 다음과 같은 로그가 있다고 가정해보자.

```text
namespace="app"
pod="api-123"
container="api"
message="database connection timeout"
```

Loki는 로그 메시지 전체를 인덱싱하지 않고, `namespace`, `pod`, `container` 같은 라벨을 중심으로 인덱싱한다.

검색할 때는 먼저 라벨로 대상을 좁힌다.

```logql
{namespace="app"}
```

그다음 특정 문자열을 포함한 로그만 필터링할 수 있다.

```logql
{namespace="app"} |= "error"
```

이 쿼리는 특정 namespace의 로그 중 `error`라는 문자열이 포함된 로그를 찾는다.

---

## LogQL

LogQL은 Loki의 쿼리 언어다.

PromQL과 비슷한 느낌으로 로그를 조회할 수 있다.

예를 들어 특정 namespace의 로그를 보고 싶다면 다음과 같이 쿼리할 수 있다.

```logql
{namespace="app"}
```

특정 문자열이 포함된 로그만 보고 싶다면 다음과 같이 작성할 수 있다.

```logql
{namespace="app"} |= "error"
```

특정 문자열을 제외할 수도 있다.

```logql
{namespace="app"} != "healthcheck"
```

JSON 로그를 파싱해서 조건을 걸 수도 있다.

```logql
{namespace="app"} | json | level="error"
```

이처럼 LogQL을 사용하면 Grafana Explore 화면에서 로그를 빠르게 검색할 수 있다.

---

## Grafana에서 메트릭과 로그를 함께 보기

Prometheus와 Loki를 모두 Grafana에 연결하면 하나의 화면에서 메트릭과 로그를 함께 볼 수 있다.

예를 들어 다음과 같은 흐름이 가능하다.

```text
Grafana 대시보드에서 에러율 증가 확인
→ 해당 시점으로 시간 범위 고정
→ Loki 로그 데이터소스로 이동
→ 같은 namespace, pod, app 라벨로 로그 검색
→ 에러 메시지 확인
```

이렇게 되면 메트릭과 로그를 따로 보는 것보다 문제 분석 속도가 빨라진다.

나중에 Tempo 같은 트레이스 도구까지 연결하면 다음 흐름도 가능해진다.

```text
메트릭
→ 로그
→ 트레이스
```

관측 가능성 도구들이 같은 라벨 체계를 공유하면, 문제를 추적하는 흐름이 훨씬 자연스러워진다.

---

## 알림이 필요한 이유

대시보드가 있어도 사람이 항상 보고 있을 수는 없다.

따라서 특정 조건이 발생했을 때 담당자에게 알려주는 알림 체계가 필요하다.

예를 들면 다음과 같은 상황이다.

```text
Pod가 계속 재시작된다.
CPU 사용률이 일정 시간 이상 높다.
메모리 사용량이 임계치를 넘었다.
에러율이 증가했다.
HTTP 5xx 비율이 높다.
Deployment의 원하는 replica 수와 ready replica 수가 다르다.
디스크 사용률이 높다.
특정 Job이 실패했다.
```

이런 조건은 PrometheusRule로 정의하고, Alertmanager를 통해 알림을 보낼 수 있다.

---

## PrometheusRule과 Alertmanager

PrometheusRule은 Prometheus가 평가할 알림 규칙을 Kubernetes 리소스로 선언하는 방식이다.

예를 들어 다음과 같은 내용을 정의할 수 있다.

```text
최근 5분 동안 에러율이 5%를 넘으면 알림 발생
Pod가 10분 이상 CrashLoopBackOff 상태면 알림 발생
CPU 사용률이 15분 이상 90% 이상이면 알림 발생
```

Alertmanager는 Prometheus에서 발생한 알림을 받아서 라우팅하고, 중복을 제거하고, 적절한 수신자에게 전달하는 역할을 한다.

흐름은 다음과 같다.

```text
Prometheus
→ PrometheusRule 평가
→ 조건 충족 시 Alert 발생
→ Alertmanager로 전달
→ 라우팅 규칙에 따라 담당자에게 알림
```

Alertmanager는 다음과 같은 기능을 제공한다.

- 알림 라우팅
- 중복 알림 제거
- 알림 그룹화
- 알림 억제
- Silence 설정
- Webhook 연동

---

## 외부 알림 채널과의 연결

Alertmanager는 다양한 방식으로 외부 알림 채널과 연동할 수 있다.

일반적으로 많이 사용하는 방식은 다음과 같다.

- Email
- Slack
- Microsoft Teams
- PagerDuty
- Opsgenie
- Webhook

특정 조직에서 카카오톡, 문자, 전화 같은 알림이 필요하다면 보통 Webhook을 활용해 중간 연동 서비스를 두는 방식으로 구성할 수 있다.

예를 들면 다음과 같다.

```text
Alertmanager
→ Webhook Receiver
→ 내부 알림 중계 서비스
→ SMS / 전화 / 메신저 알림
```

이때 내부 알림 중계 서비스는 Alertmanager에서 받은 alert payload를 조직의 알림 시스템 형식에 맞게 변환한다.

중요한 것은 Alertmanager가 모든 알림 채널을 직접 지원해야 하는 것이 아니라, Webhook을 통해 조직 내부 알림 시스템과 연결할 수 있다는 점이다.

---

## 필요한 알림 Rule 정리하기

관측 가능성을 구축할 때는 도구 설치만큼 중요한 것이 어떤 알림을 만들 것인지 정하는 일이다.

처음부터 너무 많은 알림을 만들면 노이즈가 커진다.  
반대로 너무 적게 만들면 장애를 늦게 알아차릴 수 있다.

기본적으로 검토할 수 있는 알림 Rule은 다음과 같다.

| 영역 | 알림 예시 |
|---|---|
| Pod | CrashLoopBackOff, Pending 상태 지속, 재시작 횟수 증가 |
| Deployment | 원하는 replica 수와 ready replica 수 불일치 |
| Node | NotReady 상태, CPU/Memory/Disk 사용률 임계치 초과 |
| Application | HTTP 5xx 증가, 응답 시간 p95 증가, 에러율 증가 |
| Job | Job 실패, CronJob 미실행 |
| Storage | PVC 용량 부족, 디스크 사용률 증가 |
| Network | 요청 실패율 증가, 외부 API 호출 실패 |
| Prometheus | scrape 실패, target down |
| Loki | 로그 수집 지연, ingestion 실패 |
| Thanos | Compactor 오류, Store Gateway 오류, Query 오류 |

알림 Rule은 한 번에 완성되는 것이 아니라 운영하면서 조정해야 한다.

좋은 알림은 다음 기준을 만족해야 한다.

```text
실제로 대응이 필요한가?
담당자가 원인을 추적할 수 있는 정보를 포함하는가?
너무 자주 발생하지 않는가?
장애 전에 미리 감지할 수 있는가?
업무 시간과 비업무 시간의 라우팅이 구분되는가?
```

---

## 관측 가능성과 기존 모니터링 시스템

조직마다 이미 사용 중인 서버 모니터링 시스템이나 알림 시스템이 있을 수 있다.

Prometheus, Loki, PrometheusRule, Alertmanager를 잘 구성하면 기존 서버 모니터링 에이전트가 하던 역할 중 상당 부분을 대체하거나 보완할 수 있다.

예를 들면 다음과 같다.

```text
서버 리소스 모니터링
→ node-exporter + Prometheus

Kubernetes 오브젝트 상태 모니터링
→ kube-state-metrics + Prometheus

애플리케이션 메트릭
→ /metrics + ServiceMonitor

로그 중앙 수집
→ Fluent Bit + Loki

알림
→ PrometheusRule + Alertmanager
```

다만 기존 시스템이 담당자 호출, 문자, 전화, 메신저 알림까지 담당하고 있다면 그 부분은 별도 연동이 필요하다.

즉, Prometheus와 Alertmanager는 “알림 조건을 판단하고 이벤트를 발생시키는 역할”에 강하고, 실제 조직별 통보 채널은 Webhook이나 내부 알림 중계 서비스를 통해 연결하는 방식이 자연스럽다.

---

## 운영 시 반드시 고려할 점

Prometheus, Loki, Thanos 같은 관측 가능성 스택을 운영할 때는 설치 자체보다 운영 정책이 더 중요하다.

특히 다음 항목을 반드시 고려해야 한다.

### 1. Retention 정책

메트릭과 로그를 얼마나 오래 보관할지 정해야 한다.

```text
최근 데이터는 고해상도로 보관
오래된 데이터는 다운샘플링
필요 없는 데이터는 삭제
```

보관 정책이 없으면 저장소 용량은 계속 증가한다.

---

### 2. 다운샘플링 정책

장기 저장 메트릭은 원본 해상도 그대로 모두 보관하기 어려울 수 있다.

따라서 오래된 데이터는 5분 단위, 1시간 단위처럼 낮은 해상도로 변환해 보관할 수 있다.

```text
최근 30일: raw 데이터
최근 90일: 5분 단위 데이터
최근 1년: 1시간 단위 데이터
```

---

### 3. 고가용성

운영 환경에서는 관측 가능성 스택 자체도 이중화가 필요할 수 있다.

예를 들어 다음과 같은 구성을 생각할 수 있다.

```text
Prometheus 2벌 구성
Thanos Query 이중화
Store Gateway 이중화
Loki 이중화
Alertmanager 이중화
```

특히 Thanos Compactor는 주의가 필요하다.

동일한 오브젝트 스토리지 버킷에 대해 Compactor를 여러 개 실행하면 데이터 손상 위험이 있을 수 있으므로, 중복 실행을 피해야 한다.

---

### 4. Secret 관리

S3 접근 키나 비밀번호 같은 값은 코드나 Manifest에 직접 하드코딩하면 안 된다.

Kubernetes Secret, 외부 Secret Manager, sealed secret 같은 방식을 사용해야 한다.

```text
Access Key
Secret Key
Endpoint
Bucket 정보
Webhook URL
알림 토큰
```

이런 값은 Git에 그대로 노출되지 않도록 관리해야 한다.

---

### 5. 저장소 내구성

Thanos나 Mimir를 사용해 S3 호환 오브젝트 스토리지에 장기 데이터를 저장한다면, 오브젝트 스토리지 자체의 내구성도 중요하다.

다음 항목을 확인해야 한다.

```text
복제 정책
Erasure coding
백업 정책
버킷 lifecycle 정책
접근 권한
암호화
장애 복구 절차
```

관측 가능성 데이터도 장애 분석과 리포팅에 필요한 중요한 데이터이므로, 저장소 안정성을 함께 고려해야 한다.

---

## 비유로 이해하기

Prometheus와 Thanos, S3, Loki를 비유하면 다음과 같이 이해할 수 있다.

```text
Prometheus = 사무실 책상
S3 오브젝트 스토리지 = 장기 문서 창고
Thanos = 문서관리 시스템
Loki = 사건 기록 보관함
Fluent Bit = 각 현장의 기록 수집 담당자
Grafana = 문서를 보기 좋게 보여주는 화면
Alertmanager = 이상 상황을 담당자에게 알려주는 알림 담당자
내부 레지스트리 = 외부 자재를 들여오는 반입 검사대
```

Prometheus는 최근 데이터를 빠르게 확인할 수 있는 책상과 같다.  
하지만 책상 위에 모든 문서를 계속 쌓아둘 수는 없다.

오래 보관해야 하는 메트릭은 문서 창고에 넣어야 한다.  
S3 오브젝트 스토리지가 그 창고 역할을 한다.

Thanos는 그 창고에 문서를 넣고, 색인하고, 필요할 때 다시 꺼내볼 수 있게 해주는 문서관리 시스템에 가깝다.

Loki는 각 서비스에서 발생한 사건 기록을 모아두는 보관함이고, Fluent Bit는 각 노드에서 그 기록을 수집해 보내는 역할을 한다.

Grafana는 이 모든 데이터를 보기 좋게 조회할 수 있는 화면이다.

Alertmanager는 정해진 조건을 만족하는 이상 상황이 발생했을 때 담당자에게 알려주는 역할을 한다.

---

## 관측 가능성과 GitOps

이 장을 읽으면서 관측 가능성 역시 GitOps와 잘 연결될 수 있다고 느꼈다.

모니터링 도구 설치, 대시보드, 알림 규칙, ServiceMonitor 같은 리소스도 결국 선언형으로 관리할 수 있다.

예를 들면 다음과 같은 항목들을 Git에 선언할 수 있다.

- Prometheus 설치 설정
- Grafana 대시보드
- Loki 설정
- Fluent Bit 설정
- Alertmanager 라우팅 규칙
- PrometheusRule
- ServiceMonitor
- PodMonitor
- Thanos 설정
- retention 정책
- scrape interval
- namespace별 모니터링 정책

이렇게 하면 관측 가능성 구성도 Git에 기록되고 리뷰할 수 있다.

```text
모니터링 설정 변경
→ Git에 Manifest 수정
→ Pull Request 리뷰
→ GitOps 도구가 반영
→ 실제 모니터링 스택 변경
```

즉, 관측 가능성도 “한 번 설치하고 끝”이 아니라, 애플리케이션과 함께 계속 진화하는 인프라 코드의 일부로 다뤄야 한다.

---

## AI와 함께 관측 가능성을 구축한다는 것

이번 장의 주제는 Claude Code를 활용해 관측 가능성 구성을 한 번에 구축하는 흐름이다.

여기서 중요한 점은 AI가 단순히 Helm 명령을 대신 실행해주는 것이 아니다.

AI에게 다음과 같은 작업을 맡길 수 있다.

```text
현재 클러스터 상태 확인
적절한 모니터링 스택 제안
Helm values 작성
ServiceMonitor 작성
Loki와 Fluent Bit 구성
Grafana 데이터소스 추가
대시보드 구성 안내
알림 규칙 초안 작성
Alertmanager 라우팅 구성
문서 갱신
```

하지만 관측 가능성은 운영에 직접 영향을 주는 영역이기 때문에 AI에게 모든 것을 바로 실행하게 하는 것은 위험할 수 있다.

따라서 다음 흐름이 필요하다.

```text
탐색
→ 비교
→ 설계
→ Manifest 생성
→ 리뷰
→ 적용
→ 검증
→ 문서화
```

AI는 구성 초안을 만들고 반복 작업을 줄이는 데 유용하다.  
하지만 어떤 메트릭을 볼 것인지, 어떤 로그를 수집할 것인지, 어떤 알림이 필요한지, 데이터를 얼마나 보관할지는 사람이 운영 목적에 맞게 결정해야 한다.

---

## 내가 이해한 방식

Chapter 4를 읽고 관측 가능성을 다음과 같이 이해했다.

```text
메트릭은 이상 징후를 알려준다.
로그는 문제의 원인을 찾게 해준다.
트레이스는 요청 흐름에서 병목을 보여준다.
프로파일은 코드 수준의 성능 병목을 보여준다.
알림은 이상 상황을 사람에게 전달한다.
```

그리고 Kubernetes 환경에서 관측 가능성의 기본 흐름은 다음과 같이 볼 수 있다.

```text
Prometheus = 메트릭 수집
Grafana = 시각화
Alertmanager = 알림
ServiceMonitor = scrape 대상 선언
kube-state-metrics = Kubernetes 오브젝트 상태 메트릭화
Loki = 로그 저장과 조회
Fluent Bit = 로그 수집과 전송
Thanos = Prometheus 장기 저장 확장
```

관측 가능성은 장애가 발생한 뒤에 급하게 붙이는 도구가 아니라, 시스템을 운영하기 위한 기본 인프라에 가깝다.

특히 AI와 GitOps를 함께 사용하는 환경에서는 관측 가능성이 더 중요해진다.

AI가 배포와 인프라 변경을 도와줄수록, 그 변경이 실제 시스템에 어떤 영향을 주는지 확인할 수 있어야 하기 때문이다.

---

## 더 알아볼 것

이 장을 읽고 나서 더 확인해보고 싶은 내용은 다음과 같다.

- `kube-prometheus-stack`의 주요 구성 요소
- Prometheus Operator의 역할
- ServiceMonitor와 PodMonitor의 차이
- PrometheusRule 작성 방식
- Alertmanager 알림 라우팅 방식
- Alertmanager Webhook 연동 방식
- 조직 내부 알림 시스템과 Alertmanager를 연결하는 방법
- Grafana 대시보드를 GitOps로 관리하는 방법
- Loki를 이용한 로그 수집 구조
- Fluent Bit DaemonSet 구성 방식
- LogQL 사용법
- Tempo를 이용한 트레이스 수집 구조
- Pyroscope를 이용한 프로파일링 구성
- Thanos Sidecar 구성 방식
- Thanos Compactor retention 정책
- Thanos와 Mimir의 운영 복잡도 차이
- VictoriaMetrics를 선택하기 좋은 상황
- 폐쇄망 환경에서 Helm 차트와 이미지를 반입하는 방식
- 관측 가능성 스택을 GitOps로 관리하는 방법
- 운영 환경에서 꼭 필요한 기본 알림 Rule 목록

---

## 정리

Chapter 4에서 가장 크게 남은 키워드는 다음과 같다.

- 관측 가능성은 메트릭, 로그, 트레이스가 핵심이다.
- 프로파일은 코드 수준 병목을 찾는 네 번째 요소로 볼 수 있다.
- Prometheus는 메트릭을 수집하고 Grafana는 이를 시각화한다.
- Kubernetes 환경에서는 `kube-prometheus-stack`이 관측 가능성 구축의 좋은 출발점이 될 수 있다.
- Prometheus는 Pull 방식으로 `/metrics` 엔드포인트를 scrape한다.
- PromQL은 시계열 데이터를 의미 있게 조회하기 위한 쿼리 언어다.
- ServiceMonitor는 어떤 Service를 scrape할지 선언하는 리소스다.
- kube-state-metrics는 Kubernetes 오브젝트 상태를 메트릭으로 바꿔준다.
- 프로덕션 환경에서는 Prometheus 장기 저장 전략이 필요하다.
- S3 호환 오브젝트 스토리지가 있다면 Thanos를 통해 기존 Prometheus에 장기 저장을 붙이는 구조를 검토할 수 있다.
- 초대규모 멀티테넌트 환경에서는 Mimir가 대안이 될 수 있다.
- 운영 단순성이 중요하다면 VictoriaMetrics도 선택지가 될 수 있다.
- 메트릭만으로는 원인을 알기 어렵기 때문에 중앙 로그 수집이 필요하다.
- Loki와 Fluent Bit는 Kubernetes 로그 수집을 가볍게 시작할 수 있는 조합이다.
- Alertmanager는 PrometheusRule에서 발생한 알림을 담당자에게 라우팅하는 역할을 한다.
- 관측 가능성 구성도 GitOps로 선언하고 관리할 수 있다.

이번 장은 단순히 모니터링 도구를 설치하는 내용이 아니었다.

시스템을 운영한다는 것은 애플리케이션을 배포하는 것에서 끝나지 않는다.  
배포 이후 시스템이 어떤 상태인지 계속 관찰할 수 있어야 한다.

특히 AI와 GitOps를 활용해 인프라 변경이 더 쉬워지는 시대에는 관측 가능성이 더 중요해진다.

변경이 쉬워질수록 다음 질문에 빠르게 답할 수 있어야 한다.

```text
변경 후 에러율이 증가했는가?
응답 시간이 느려졌는가?
특정 Pod나 Node에 부하가 몰렸는가?
배포 이후 어떤 로그가 증가했는가?
문제가 어느 서비스 구간에서 발생했는가?
알림이 적절한 담당자에게 전달되었는가?
```

결국 관측 가능성은 운영의 부가 기능이 아니라, 안전한 자동화를 위한 전제 조건이다.

AI가 인프라를 더 많이 다루게 될수록, 우리는 더 잘 관찰할 수 있는 시스템을 만들어야 한다.

> 자동화가 빨라질수록  
> 관측 가능성은 더 중요해진다.
