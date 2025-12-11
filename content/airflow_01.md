---
title: "DAG 정리의 기술: EmptyOperator vs TaskGroup"
description: "This is a description"
date: 2025-12-11
draft: false
tags:
  - Airflow
  - Orchestration
  - DAG Design
---

## 세 줄 요약

- **EmptyOperator**는 실제 스케줄러가 인식하는 **태스크(Task, 논리적 그룹핑)**입니다.
- **TaskGroup**은 스케줄러에게는 보이지 않는 **UI 편의 기능(시각적 그룹핑)**입니다.
- **흐름 제어나 동기화(Join)**가 필요하면 EmptyOperator, **화면 정리 및 가독성**이 목적이라면 TaskGroup을 사용하세요.

---

## 1. 문제 정의

DAG를 작성하다 보면 태스크 간 의존성(Dependency)이 복잡해져 '스파게티 그래프'가 되기 쉽습니다.  
이를 정리하기 위해 `EmptyOperator`(구 DummyOperator)와 `TaskGroup`을 사용하지만, 둘의 기술적 차이를 명확히 구분하지 않고 혼용하는 경우가 많습니다.

---

## 2. 기술적 차이

| 구분            | EmptyOperator                                  | TaskGroup                      |
| --------------- | ---------------------------------------------- | ------------------------------ |
| **정체**        | BaseOperator를 상속받은 실제 태스크            | Operator가 아님(추상화 계층)   |
| **동작**        | 연산 없이 실행, 상태(Success/Fail) 가짐        | 실행 시 Flatten되어 태스크만 전달됨 |
| **Scheduler 인식** | **있음(논리적 노드, 독립 작업)**             | **없음(UI만 그룹 박스 존재)**   |
| **주 용도**     | 다수 태스크 Join(동기화), 시작/종료 마커        | UI 가독성 향상, 태스크 묶어서 접기/펼치기 |

### ✔️ EmptyOperator (The Logic)

- **정체:** 실제 태스크 (BaseOperator 상속)
- **동작:** 연산 X, 실행 슬롯 및 상태 가짐
- **용도:**
  - 여러 Upstream 태스크 완료 후 Downstream으로 한 번에 넘기는 Join 포인트 생성  
  - DAG 시작/종료 마커

### ✔️ TaskGroup (The View)

- **정체:** Operator가 아닌, DAG 구조화를 위한 추상화 계층 (Airflow 2.0 도입)
- **동작:** 최종 실행 시에는 Group개념이 사라지고, 태스크만 평평하게(Flatten) 스케줄러에 등록
- **용도:**
  - UI에서 많은 태스크를 논리적으로 묶고 접었다 펼치기
  - 관리 편의성 및 가독성 향상

---

## 3. 코드 비교 (Implementation)

### 🎯 Case 1: `EmptyOperator`로 흐름 제어 (Fan-in)

여러 작업이 끝나는 지점을 명시적으로 하나로 모으고 싶을 때 사용합니다.

```python
from airflow.operators.empty import EmptyOperator

# 실제 작업들
t1 = ...
t2 = ...

# Join 지점 생성
join_task = EmptyOperator(task_id='join_tasks')

[t1, t2] >> join_task
```

### 🎯 Case 2: `TaskGroup`으로 화면 정리

전처리 관련 태스크 여러 개를 하나의 박스로 묶어 가독성을 높이고 싶을 때 사용합니다.

```python
from airflow.utils.task_group import TaskGroup

with TaskGroup("preprocess_group") as tg:
    t1 >> t2 >> t3
    # 스케줄러는 t1, t2, t3 각각 인식
    # UI에서는 'preprocess_group'이라는 박스로 보임
```

---

## 4. 결론 및 참고자료

- **그래프의 선(Edge)**을 줄이고 싶다면 → `EmptyOperator`
- **그래프의 노드(Node) 개수**를 시각적으로 줄여 보이고 싶다면 → `TaskGroup`

### Reference

- [Airflow Docs: Dags](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html)
- [Airflow Docs: Grouping Tasks](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html#taskgroups)
- [sparkcodehub.com/airflow/operators/empty-operator](https://sparkcodehub.com/airflow/operators/empty-operator)