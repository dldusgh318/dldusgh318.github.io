---
title: "Grafana Cloud로 운영용 대시보드와 Discord 알림 만들기"
date: 2026-09-07 16:20:00 +0900
project: chungbazi
categories: [Infra, 모니터링]
tags: [grafana-cloud, promql, logql, alerting, discord]
series: Grafana Cloud로 모니터링 비용 줄이기
media_subpath: /assets/img/posts/grafana-cloud
render_with_liquid: false
image:
  path: 05-dashboard.jpg
  alt: ChungBazi Backend 운영 대시보드
---

> 2편까지 서버·애플리케이션 메트릭과 로그를 Grafana Cloud로 모았다. 이번 편에서는 운영자가 한 화면에서 상태를 볼 수 있는 대시보드를 만들고, 장애를 Discord로 받도록 알림을 구성한다.
{: .prompt-info }

## 7. 애플리케이션 전용 Dashboard 구성

대시보드 이름은 `ChungBazi Backend - Production`으로 정하고, 운영자가 서비스 상태·트래픽·오류·자원·로그를 한 화면에서 확인할 수 있도록 구성했다.

![ChungBazi Backend 대시보드](05-dashboard.jpg)

### Dashboard 배치

- **1행:** Application Status / JVM Heap Usage / Application CPU Usage
- **2행:** API Requests per Second / API 5xx Error Rate
- **3행:** Average API Response Time / Database Connection Pool
- **4행:** Application Logs
- Grafana 그리드는 24칸이므로 1행의 Gauge 및 Stat 패널은 각각 8칸 너비로 배치했다.

### Application Status

애플리케이션의 Prometheus scrape 성공 여부를 표시한다.

```text
up{job="chungbazi-backend", environment="production"}
```

| 설정 | 값 |
| --- | --- |
| Visualization | Stat |
| Value mapping | `1 → UP`, `0 → DOWN` |
| Min / Max | `0` / `1` |

### API Requests per Second

Actuator 요청을 제외한 초당 API 요청 수를 표시한다.

```text
sum(
  rate(
    http_server_requests_seconds_count{
      job="chungbazi-backend",
      environment="production",
      uri!~"/actuator.*"
    }[$__rate_interval]
  )
)
```

| 설정 | 값 |
| --- | --- |
| Visualization | Time series |
| Legend | `Total requests` |
| Unit | requests/sec (rps) |

### API 5xx Error Rate

전체 API 요청 중 5xx 응답의 비율을 표시한다. 5xx 시계열이 아직 없어도 `0`으로 표현되도록 `or vector(0)`을 사용했다.

```text
100 *
(
  sum(
    rate(
      http_server_requests_seconds_count{
        job="chungbazi-backend",
        environment="production",
        uri!~"/actuator.*",
        status=~"5.."
      }[$__rate_interval]
    )
  )
  or vector(0)
)
/
clamp_min(
  sum(
    rate(
      http_server_requests_seconds_count{
        job="chungbazi-backend",
        environment="production",
        uri!~"/actuator.*"
      }[$__rate_interval]
    )
  ),
  0.000001
)
```

| 설정 | 값 |
| --- | --- |
| Visualization | Time series |
| Legend | `5xx error rate` |
| Unit | Percent (0-100) |
| Min | `0` |

### Average API Response Time

요청 처리 시간의 합을 요청 수로 나눠 평균 응답시간을 계산한다. Actuator의 초 단위 값을 밀리초로 변환하기 위해 `1000`을 곱했다.

```text
1000 *
sum(
  rate(
    http_server_requests_seconds_sum{
      job="chungbazi-backend",
      environment="production",
      uri!~"/actuator.*"
    }[$__rate_interval]
  )
)
/
clamp_min(
  sum(
    rate(
      http_server_requests_seconds_count{
        job="chungbazi-backend",
        environment="production",
        uri!~"/actuator.*"
      }[$__rate_interval]
    )
  ),
  0.000001
)
```

| 설정 | 값 |
| --- | --- |
| Visualization | Time series |
| Legend | `Average` |
| Unit | milliseconds (ms) |
| Min | `0` |

### JVM Heap Usage

JVM Heap 최대 크기 대비 현재 사용량을 Gauge로 표시한다.

```text
100 *
sum(
  jvm_memory_used_bytes{
    job="chungbazi-backend",
    environment="production",
    area="heap"
  }
)
/
sum(
  jvm_memory_max_bytes{
    job="chungbazi-backend",
    environment="production",
    area="heap"
  }
)
```

| 설정 | 값 |
| --- | --- |
| Visualization | Gauge |
| Query type | Instant |
| Unit | Percent (0-100) |
| Threshold | `70` 노랑, `85` 빨강 |

### Application CPU Usage

EC2 전체가 아닌 Spring Boot JVM 프로세스의 CPU 사용률을 표시한다.

```text
100 *
process_cpu_usage{
  job="chungbazi-backend",
  environment="production"
}
```

| 설정 | 값 |
| --- | --- |
| Visualization | Gauge |
| Query type | Instant |
| Unit | Percent (0-100) |
| Threshold | `70` 노랑, `85` 빨강 |

### Database Connection Pool

HikariCP의 현재 사용 중인 연결 수와 최대 연결 수를 한 패널에서 비교한다.

**Query A — Active**

```text
sum(
  hikaricp_connections_active{
    job="chungbazi-backend",
    environment="production"
  }
)
```

**Query B — Max**

```text
sum(
  hikaricp_connections_max{
    job="chungbazi-backend",
    environment="production"
  }
)
```

| 설정 | 값 |
| --- | --- |
| Visualization | Time series |
| Legends | `Active`, `Max` |
| Unit | short |
| Min | `0` |

### Application Logs

Loki에서 운영 Spring Boot 컨테이너 로그를 조회한다.

```text
{service_name="chungbazi-backend", environment="production"} | json
```

| 설정 | 값 |
| --- | --- |
| Data source | Loki |
| Visualization | Logs |
| Order | Newest first |
| 옵션 | Show time, Wrap lines 활성화 |
| Deduplication | None |

INFO 이상만 확인할 때는 다음 쿼리를 사용할 수 있다.

```text
{service_name="chungbazi-backend", environment="production"}
| json
| level=~"INFO|WARN|ERROR"
```

## 8. Alert 알림 구성

Discord Contact point를 생성하고 이름을 `chungbazi-prod-discord`로 지정했다. Webhook URL은 자격 증명이므로 문서나 Git 저장소에 기록하지 않는다.

### Discord Contact point

1. Discord 알림 채널에서 Webhook을 생성한다.
2. Grafana Cloud의 **Alerting → Contact points**에서 Discord Integration을 추가한다.
3. Contact point 이름을 `chungbazi-prod-discord`로 지정한다.
4. Test notification으로 Discord 수신 여부를 확인한다.
5. Webhook URL은 GitHub, Notion, 로그, 화면 캡처에 노출하지 않는다.

### 알림 규칙 4종 한눈에 보기

| Rule | 조건 | Pending | No data | 이유 |
| --- | --- | --- | --- | --- |
| Application Down | `up` 마지막 값 < `1` | 2m (Keep firing 5m) | Alerting | 애플리케이션뿐 아니라 서버 또는 Alloy 장애로 메트릭이 사라진 상황도 감지한다. |
| 5xx Errors | 최근 5분 5xx > `2`건 (3건 이상) | 1m | Normal | 트래픽이 적은 단계에서는 오류율보다 오류 건수 기준이 불필요한 변동을 줄인다. |
| High JVM Heap | Heap > `85%` | 5m | Normal | GC 직전의 순간적인 증가를 제외하고 장시간 지속되는 메모리 압박을 감지한다. |
| DB Pool Saturation | 사용 커넥션 > `80%` | 5m | Normal | 커넥션 고갈로 요청이 대기하거나 실패하기 전에 느린 쿼리와 커넥션 누수를 확인한다. |

모든 규칙의 Evaluation interval은 `1m`, Error 상태는 Alerting으로 설정했다.

### Application Down

```text
up{job="chungbazi-backend", environment="production"}
```

- Rule name: `ChungBazi Production - Application Down`
- No data를 Alerting으로 둔 이유: 메트릭 자체가 사라진 상황이 가장 위험한 장애이기 때문이다.

### 5xx Errors

```text
sum(
  increase(
    http_server_requests_seconds_count{
      job="chungbazi-backend",
      environment="production",
      uri!~"/actuator.*",
      status=~"5.."
    }[5m]
  )
)
or vector(0)
```

- Rule name: `ChungBazi Production - 5xx Errors`

### High JVM Heap

```text
100 *
sum(
  jvm_memory_used_bytes{
    job="chungbazi-backend",
    environment="production",
    area="heap"
  }
)
/
sum(
  jvm_memory_max_bytes{
    job="chungbazi-backend",
    environment="production",
    area="heap"
  }
)
```

- Rule name: `ChungBazi Production - High JVM Heap`

### DB Pool Saturation

```text
100 *
sum(
  hikaricp_connections_active{
    job="chungbazi-backend",
    environment="production"
  }
)
/
clamp_min(
  sum(
    hikaricp_connections_max{
      job="chungbazi-backend",
      environment="production"
    }
  ),
  1
)
```

- Rule name: `ChungBazi Production - DB Pool Saturation`

## 현재 수집 범위

- [x] 운영 EC2 CPU
- [x] 운영 EC2 메모리
- [x] 파일시스템 및 디스크
- [x] 네트워크
- [x] systemd journal 로그
- [x] Linux Server 기본 Dashboard
- [x] Linux Server 기본 Alert rules
- [x] Spring Boot Actuator Prometheus 메트릭
- [x] Docker 애플리케이션 JSON 로그
- [x] 애플리케이션 전용 Dashboard
- [x] Discord Alert 알림 채널
- [x] 애플리케이션 핵심 Alert rules 4종

## 핵심 회고

> Grafana Cloud와 Alloy의 연결 성공, 데이터 수집 설정, Dashboard 설치는 서로 다른 단계다. Alloy 프로세스가 실행 중이라는 사실만으로 메트릭이 전송된다고 판단하면 안 된다. readiness, 설정 문법, Fleet 요청 성공 여부, 실제 수집 메트릭까지 단계별로 검증해야 한다.

## 참고 자료

- [Grafana가 바꾸는 Observability의 기준](https://rockplace.tistory.com/321)
- [Grafana Cloud Linux Server Integration](https://grafana.com/docs/grafana-cloud/observe-and-act/monitor-infrastructure/integrations/integration-reference/integration-linux-node/)
