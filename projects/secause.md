---
layout: page
title: SeCause
permalink: /projects/secause/
project: secause
---

{% include project-header.html %}

## 어떤 프로젝트인가요

GitHub 저장소의 코드를 AI로 분석해 보안 취약점을 찾아 주는 졸업 프로젝트입니다. 백엔드를 맡아 분석 요청을 받고 AI 분석 서버(FastAPI)로 넘기는 흐름을 만들었습니다.

## 핵심 문제 해결

### Spring 비동기 전달과 Redis/RQ 작업 큐로 분석 접수·실행 분리

분석 요청을 DB에 저장하고, 커밋된 뒤 전용 Executor에서 FastAPI로 전달하도록 구현했습니다.
FastAPI는 Redis/RQ에 작업을 넣은 뒤 `202`와 job ID를 바로 돌려주고, 별도 워커가 실제 분석을 처리하도록 접수와 실행을 분리했습니다. 큐 적재에 실패하면 `503`을 반환합니다.

| 지표 | 동기 | 비동기 |
| --- | --- | --- |
| 요청 p95 | 3,189ms | 221ms |

### 비동기로 분리한 구간의 실패를 실험으로 찾기

응답이 빨라졌다고 끝내지 않고, Spring에서 FastAPI로 넘기는 구간만 따로 재현해 봤습니다.

- **포화 실험:** 140건을 한꺼번에 넣었더니 36건이 전달되지 못하고 `PENDING`으로 남았습니다.
- **재시작 실험:** 25건을 처리하는 도중 서버를 종료하자, 재시작 뒤에도 `PENDING` 23건과 `IN_PROGRESS` 2건이 그대로 남았습니다.

이를 바탕으로, RQ에 넣기 전 전달 실패를 복구할 수 있도록 **작업 기록 영속화 · 재처리 · 멱등성** 설계를 제안했습니다.

> Spring 전달 구간 로컬 실험: H2 · 모의 FastAPI 수락 3초 · 동시 100건. RQ나 분석 자체의 성능을 측정한 것은 아닙니다.
{: .prompt-info }

{% include project-posts.html %}
