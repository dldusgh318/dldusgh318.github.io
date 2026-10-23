---
title: "Spring Boot 메트릭과 Docker 로그를 Grafana Cloud로 보내기"
date: 2026-09-07 16:10:00 +0900
project: chungbazi
categories: [Infra, 모니터링]
tags: [grafana-cloud, alloy, actuator, loki, docker]
series: Grafana Cloud로 모니터링 비용 줄이기
media_subpath: /assets/img/posts/grafana-cloud
render_with_liquid: false
---

> 1편에서 운영 EC2의 시스템 메트릭과 journal 로그를 Grafana Cloud로 보냈다. 이번 편에서는 같은 Alloy로 **Spring Boot Actuator 메트릭**과 **Docker 애플리케이션 JSON 로그**까지 수집한다.
{: .prompt-info }

## 5. Spring Boot Actuator 메트릭 수집

운영 애플리케이션의 `/actuator/prometheus`를 Alloy가 로컬에서 직접 스크레이프하도록 구성했다. 관리 포트 9090은 외부 보안 그룹에 열지 않고 EC2 내부의 `127.0.0.1`로만 접근한다.

### 엔드포인트 확인

```bash
curl --fail --silent http://127.0.0.1:9090/actuator/prometheus | head
```

### Alloy 설정 백업

```bash
sudo cp /etc/alloy/config.alloy /etc/alloy/config.alloy.before-spring
```

### Alloy 스크레이프 설정

```hcl
prometheus.scrape "chungbazi_backend" {
  targets = [{
    __address__      = "127.0.0.1:9090",
    __metrics_path__ = "/actuator/prometheus",
    job              = "chungbazi-backend",
    environment      = "production",
    instance         = constants.hostname,
  }]

  forward_to      = [prometheus.remote_write.metrics_service.receiver]
  scrape_interval = "30s"
  scrape_timeout  = "10s"
}
```

수집한 메트릭은 기존 Grafana Cloud Prometheus remote write 파이프라인으로 전달한다.

### 검증

```bash
sudo alloy validate /etc/alloy/config.alloy
sudo systemctl restart alloy
sudo systemctl status alloy --no-pager
```

Grafana Explore에서 다음 PromQL 결과가 `1`이면 수집이 정상이다.

```text
up{job="chungbazi-backend", environment="production"}
```

이제 JVM 메모리·GC·스레드, HTTP 요청 수·응답 시간 등 Spring Boot/Micrometer 메트릭을 조회할 수 있다.

## 6. Docker Spring Boot JSON 로그 수집

운영 서버에서 실행 중인 애플리케이션 컨테이너의 JSON 로그만 Alloy가 읽어 Grafana Cloud Loki로 전송하도록 구성했다.

### Docker 소켓 접근 권한

Alloy 서비스 계정이 Docker 컨테이너를 탐색할 수 있도록 `docker` 그룹에 추가했다.

```bash
sudo usermod -aG docker alloy
sudo systemctl restart alloy
sudo -u alloy docker ps --format 'table {{.Names}}\t{{.Status}}'
```

> `docker` 그룹은 사실상 root에 준하는 권한이다. Alloy 계정 외에는 추가하지 않도록 주의한다.
{: .prompt-warning }

### Alloy 설정 백업

```bash
sudo cp /etc/alloy/config.alloy /etc/alloy/config.alloy.before-docker-logs
```

### 컨테이너 탐색 및 로그 전달 설정

```hcl
discovery.docker "chungbazi" {
  host             = "unix:///var/run/docker.sock"
  refresh_interval = "5s"
}

discovery.relabel "chungbazi_app_logs" {
  targets = discovery.docker.chungbazi.targets

  rule {
    source_labels = ["__meta_docker_container_name"]
    regex         = "/chungbazi-app"
    action        = "keep"
  }

  rule {
    target_label = "service_name"
    replacement  = "chungbazi-backend"
  }

  rule {
    target_label = "environment"
    replacement  = "production"
  }

  rule {
    target_label = "instance"
    replacement  = constants.hostname
  }

  rule {
    target_label = "container"
    replacement  = "chungbazi-app"
  }
}

loki.source.docker "chungbazi_app" {
  host             = "unix:///var/run/docker.sock"
  targets          = discovery.relabel.chungbazi_app_logs.output
  forward_to       = [loki.process.chungbazi_app.receiver]
  refresh_interval = "5s"
}

loki.process "chungbazi_app" {
  stage.json {
    expressions = {
      level      = "level",
      requestId  = "requestId",
      workflowId = "workflowId",
      taskName   = "taskName",
      method     = "method",
      uri        = "uri",
      status     = "status",
      durationMs = "durationMs",
    }
  }

  stage.labels {
    values = {
      level = "",
    }
  }

  stage.structured_metadata {
    values = {
      requestId  = "",
      workflowId = "",
      taskName   = "",
      method     = "",
      uri        = "",
      status     = "",
      durationMs = "",
    }
  }

  forward_to = [loki.write.grafana_cloud_loki.receiver]
}
```

> 컨테이너 이름(`/chungbazi-app`)과 컴포넌트 이름(`chungbazi_app_logs`, `chungbazi_app`)은 각자 환경에 맞게 바꾸면 된다. 단, 위아래에서 참조하는 이름이 서로 일치해야 한다.
{: .prompt-tip }

### 라벨 설계

검색 성능과 Loki의 카디널리티 비용을 고려해 값의 종류가 적고 고정적인 항목만 라벨로 사용했다.

- 고정 라벨: `service_name`, `environment`, `instance`, `container`
- 로그 레벨 라벨: `level`
- Structured metadata: `requestId`, `workflowId`, `taskName`, `method`, `uri`, `status`, `durationMs`

요청마다 값이 달라지는 `requestId` 등을 인덱스 라벨로 만들지 않아 불필요한 고카디널리티를 방지했다.

### 검증

```bash
sudo alloy validate /etc/alloy/config.alloy
sudo systemctl restart alloy
sudo systemctl status alloy --no-pager
```

애플리케이션 요청을 한 번 발생시킨 뒤 Grafana Explore에서 다음 LogQL로 확인한다.

```text
{service_name="chungbazi-backend", environment="production"}
```

---

이제 서버·애플리케이션 메트릭과 애플리케이션 로그가 모두 Grafana Cloud에 모였다. 다음 편에서는 이 데이터로 운영용 대시보드를 만들고, Discord로 알림을 받도록 구성한다.
