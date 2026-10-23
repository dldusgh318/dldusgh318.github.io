---
title: "모니터링 서버 없이 Grafana Cloud + Alloy로 모니터링 구축하기"
date: 2026-09-07 16:00:00 +0900
project: chungbazi
categories: [Infra, 모니터링]
tags: [grafana-cloud, alloy, prometheus, loki, ec2]
series: Grafana Cloud로 모니터링 비용 줄이기
media_subpath: /assets/img/posts/grafana-cloud
---

> 별도의 모니터링 서버를 운영하지 않고, 운영 EC2에는 경량 수집기인 **Grafana Alloy**만 설치했다. Alloy가 시스템 메트릭과 로그를 수집해 Grafana Cloud의 Prometheus·Loki로 전송하고, 시각화와 알림은 Grafana Cloud가 담당한다.
{: .prompt-info }

## 도입 배경

서비스 출시를 앞두고 애플리케이션 서버 외에 Prometheus, Loki, Grafana를 직접 운영하면 EC2 한 대가 추가로 필요하고 관리 비용도 늘어난다. 소규모 서비스의 초기 트래픽과 프리티어 환경을 고려해 다음 구조를 선택했다.

```text
운영 EC2
├─ Nginx
├─ Spring Boot 컨테이너
├─ Redis 컨테이너
└─ Grafana Alloy
      ├─ Linux 시스템 메트릭 수집
      ├─ systemd journal 로그 수집
      └─ Grafana Cloud로 전송
             ├─ Prometheus 호환 메트릭 저장
             ├─ Loki 로그 저장
             ├─ Dashboard
             └─ Alert
```

### 이 구조를 선택한 이유

- 모니터링 전용 EC2 비용이 발생하지 않는다.
- Prometheus와 Loki의 저장 용량·백업·업데이트를 직접 관리하지 않아도 된다.
- 운영 EC2에는 약 50~90MB 수준의 Alloy 프로세스만 추가된다.
- 이후 Spring Boot Actuator 메트릭과 Docker 로그를 같은 Alloy로 확장할 수 있다.

## 사전 준비

- Grafana Cloud Stack 생성
- 배포 리전은 운영 EC2가 있는 서울과 가까운 **Japan** 선택
- 운영 EC2: Ubuntu, AMD64
- Spring Boot Actuator 포트는 외부에 공개하지 않고 `127.0.0.1:9090`으로만 노출

## 1. Grafana Alloy 설치

Grafana 공식 APT 저장소를 추가한다.

```bash
sudo apt-get update
sudo apt-get install -y gpg wget

sudo mkdir -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/grafana.asc \
  https://apt.grafana.com/gpg-full.key
sudo chmod 644 /etc/apt/keyrings/grafana.asc

echo "deb [signed-by=/etc/apt/keyrings/grafana.asc] https://apt.grafana.com stable main" \
  | sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get install -y alloy
```

설치 후 서비스를 활성화한다.

```bash
alloy --version
sudo systemctl enable --now alloy
sudo systemctl status alloy --no-pager
```

기본 설정 파일 경로는 다음과 같다.

```text
/etc/alloy/config.alloy
```

## 2. Grafana Cloud와 Alloy 연결

Grafana Cloud에서 다음 경로로 이동했다.

```text
Connections
→ Add new connection
→ Linux Server
```

![플랫폼 선택](01-platform.png)

Ubuntu는 **Debian**, `x86_64`는 **Amd64**로 선택했다.

![API 토큰 생성](02-token.png){: width="420" .normal }
![설치 명령 (마스킹)](03-install-masked.png){: width="420" .normal }

전용 API 토큰을 만들고 Grafana에서 생성한 설치 명령을 운영 EC2에서 실행했다. 토큰이 셸 기록에 남지 않도록 실행 전후로 history를 제어했다.

```bash
set +o history

# Grafana Cloud에서 복사한 설치 명령 실행

set -o history
```

> API 토큰은 `glc_...` 형태이며 생성 직후 한 번만 표시된다. 토큰 원문, 설치 명령, `/etc/systemd/system/alloy.service.d/env.conf` 내용은 GitHub·블로그·채팅에 올리지 않는다. (위 캡처의 설치 명령도 같은 이유로 가렸다.)
{: .prompt-warning }

## 3. Alloy 상태 검증

서비스 실행 여부만으로는 Grafana Cloud 연결 성공을 확신할 수 없다. 다음 세 단계를 각각 확인했다.

### 프로세스와 readiness

```bash
sudo systemctl status alloy --no-pager
curl -i http://127.0.0.1:12345/-/ready
```

정상 응답:

```text
HTTP/1.1 200 OK
Alloy is ready.
```

### 설정 문법

```bash
sudo alloy validate /etc/alloy/config.alloy
```

오류 출력이 없으면 문법이 정상이다.

### Fleet 원격 설정 연결

```bash
curl -s http://127.0.0.1:12345/metrics \
  | grep -E \
    'remotecfg_(load_attempts_total|load_failures_total|last_load_successful)'
```

최종 결과:

```text
remotecfg_last_load_successful 1
remotecfg_load_attempts_total 2
remotecfg_load_failures_total 0
```

즉, Fleet 원격 설정 요청이 성공했고 실패는 없었다.

## 4. Linux Server 수집 설정 적용

Grafana Cloud 연결만 구성하면 Alloy가 살아 있는 것은 확인할 수 있지만, CPU·메모리·디스크 메트릭이 자동으로 생기지는 않는다.

![Linux Server Integration 스니펫](04-integration.png)

Linux Server Integration에서 제공하는 Alloy 스니펫을 `/etc/alloy/config.alloy` 맨 아래에 추가했다.

주요 컴포넌트의 역할은 다음과 같다.

| 컴포넌트 | 역할 |
| --- | --- |
| `prometheus.exporter.unix` | CPU, 메모리, 디스크, 네트워크 등 Linux 메트릭 수집 |
| `prometheus.scrape` | 수집된 메트릭 스크레이프 |
| `prometheus.relabel` | 필요한 메트릭만 선별해 전송 |
| `loki.source.journal` | systemd journal 로그 수집 |
| `prometheus.remote_write` | Grafana Cloud Metrics로 전송 |
| `loki.write` | Grafana Cloud Logs로 전송 |

수정 후 반드시 문법을 검사하고 Alloy를 재시작한다.

```bash
sudo alloy validate /etc/alloy/config.alloy
sudo systemctl restart alloy
sudo systemctl status alloy --no-pager
```

Linux Server 연결 테스트가 성공한 뒤 **Install dashboards and alerts**를 실행했다.

## 트러블슈팅 모음

### 트러블슈팅 1: Alloy는 실행 중인데 연결 테스트 실패

**증상**

Grafana 화면에서 다음 오류가 발생했다.

```text
Oops! Something went wrong.
```

그러나 로컬 readiness와 설정 문법은 정상이었다.

```text
Alloy is ready.
```

**진단**

Alloy 내부 메트릭을 확인하니 Fleet 요청 횟수가 0이었다.

```text
remotecfg_load_attempts_total 0
remotecfg_last_load_successful 0
```

HTTPS 연결도 생성되지 않았다.

```bash
sudo ss -tpn | grep alloy | grep ':443'
```

실제 실행 설정 경로는 정상임을 확인했다.

```bash
sudo tr '\0' '\n' < /proc/$(pidof alloy)/cmdline
sudo grep -E '^(CONFIG_FILE|CUSTOM_ARGS)=' /etc/default/alloy
```

**원인과 해결**

설치 스크립트가 설정 파일을 반영한 시점과 Alloy 프로세스 재시작 시점이 맞지 않아, 실행 중인 프로세스가 변경 전 설정을 유지하고 있었다.

수동으로 서비스를 재시작했다.

```bash
sudo systemctl restart alloy
sleep 65
```

재확인 결과:

```text
remotecfg_load_attempts_total 1
remotecfg_load_failures_total 0
remotecfg_last_load_successful 1
```

Fleet 연결이 정상화됐다.

### 트러블슈팅 2: 새로고침 후 토큰 입력 상태 소실

Grafana API 토큰은 생성 직후 한 번만 표시된다. 연결 마법사 페이지를 새로고침하면서 입력 상태가 사라졌다.

토큰은 systemd drop-in 파일에 저장되어 있었지만, 화면에는 토큰 이름이 아니라 실제 `glc_...` 값을 입력해야 했다.

여기서 배운 점:

- 토큰 생성 직후 안전한 비밀 관리 도구에 보관한다.
- 연결 마법사를 완료하기 전에는 페이지를 새로고침하지 않는다.
- 토큰 이름과 토큰 실제 값은 서로 다르다.
- 토큰이 노출되었다면 즉시 폐기하고 새로 발급한다.

### 트러블슈팅 3: Fleet 연결은 성공했는데 Metrics가 없음

**증상**

```text
No metrics were found for this integration.
```

Fleet 지표는 정상이었다.

```text
remotecfg_last_load_successful 1
remotecfg_load_failures_total 0
```

**원인**

Fleet 연결 설정과 Linux 메트릭 수집 설정은 별개였다.

```text
Grafana Cloud 연결 ✅
Linux node_exporter 수집 설정 ❌
```

`prometheus.remote_write` 목적지는 존재했지만, 전송할 CPU·메모리·디스크 메트릭을 만드는 `prometheus.exporter.unix` 및 `prometheus.scrape` 설정을 아직 추가하지 않았다.

**해결**

Linux Server Integration에서 제공하는 전체 Alloy 스니펫을 설정 파일에 추가하고 검증·재시작했다.

```bash
sudo alloy validate /etc/alloy/config.alloy
sudo systemctl restart alloy
```

이후 연결 테스트가 성공했고 Linux Server Dashboard에서 운영 EC2의 데이터가 표시됐다.

---

여기까지가 서버 수준의 수집이다. 다음 편에서는 같은 Alloy로 Spring Boot Actuator 메트릭과 Docker 애플리케이션 로그까지 수집한다.

## 참고 자료

- [Grafana Alloy 공식 Linux 설치 문서](https://grafana.com/docs/alloy/latest/set-up/install/linux/)
- [Grafana Cloud Linux Server Integration](https://grafana.com/docs/grafana-cloud/observe-and-act/monitor-infrastructure/integrations/integration-reference/integration-linux-node/)
- [Grafana Cloud, Kubernetes 모니터링 비용 최적화: Free Tier로 소규모 클러스터 운영하기](https://successisnotfaraway.tistory.com/147)
- [ECS(EC2) 환경에서 Grafana Cloud로 모니터링 구축하기 (Prometheus + Loki)](https://velog.io/@pp8817/%EC%9E%A5%EC%95%A0-%EB%8C%80%EC%9D%91%EC%9D%84-%EC%9C%84%ED%95%9C-Grafana-Cloud-%EB%AA%A8%EB%8B%88%ED%84%B0%EB%A7%81-%EA%B5%AC%EC%B6%95)
