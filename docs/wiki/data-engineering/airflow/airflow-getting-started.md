---
title: "Airflow Getting Started"
parent: Airflow
grand_parent: Data Engineering
permalink: /de/airflow/getting-started
---

## 1. Airflow란?

> **Airflow = 작업들을 자동으로 순서대로 실행해주는 스케줄러로 작업의 순서와 실행을 관리함**

---

## 2. 핵심 용어 4가지

### 2.1 DAG (대그) - 작업 설계도

**DAG = Directed Acyclic Graph**

> 💡 **DAG = "A 하고 → B 하고 → C 해" 이런 작업 흐름을 정의한 파일**
**예시:** 데이터 수집 → 데이터 정제 → 분석 → 리포트 생성

### 2.2 Task (태스크) - 실제 할 일

> ⚙️ **Task = DAG 안에 있는 각각의 작업 하나**
위 예시에 나온 "데이터 수집", "데이터 정제", "분석", "리포트 생성" 같은 개별 단계를 의미

### 2.3 Operator (오퍼레이터) - 작업 유형

Task를 **어떤 방식으로** 실행할지 정하는 것:

| Operator | 용도 | 쉬운 설명 |
|----------|------|-----------|
| BashOperator | 쉘 명령어 실행 | 터미널에 명령 치는 것 |
| PythonOperator | Python 함수 실행 | Python 코드 돌리기 |
| EmailOperator | 이메일 발송 | 자동 메일 보내기 |
| SQLOperator | SQL 쿼리 실행 | DB에서 데이터 가져오기 |

> BaseOperator 를 상속받아 CustomOperator를 만들 수 있음 (웬만하면 PythonOperator로 커버가 가능해서 많이 사용되진 않음)

### 2.4 Schedule (스케줄) - 언제 실행?

> ⏰ **Schedule = DAG을 언제 실행할지 정하는 것**

자주 쓰는 스케줄 표현:

| 표현 | 의미 |
|------|------|
| `@daily` | 매일 자정 |
| `@hourly` | 매시간 |
| `@weekly` | 매주 일요일 자정 |
| `0 9 * * *` | 매일 오전 9시 (cron) |

---

## 3. Airflow 구조

### 주요 구성 요소

| 구성요소 | 역할 | 비유 |
|----------|------|------|
| **Scheduler** | DAG을 모니터링하고 실행 시점이 되면 Worker에게 작업 할당 | 회사의 관리자/매니저 |
| **Worker** | 실제 Task를 실행하는 역할 | 일 하는 직원 |
| **Web Server** | UI 제공, DAG 상태 모니터링 | 대시보드/모니터 |
| **Metadata DB** | DAG, Task 상태, 실행 기록 저장 | 기록 보관소 |

### 작동 흐름

1. 개발자가 DAG 파일(.py)을 작성해서 dags 폴더에 넣음
2. **Scheduler가 DAG 파일을 읽고 파싱함**
3. 스케줄 시간이 되면 Scheduler가 Task를 큐에 넣음
4. Worker가 큐에서 Task를 가져와 실행
5. 결과가 Metadata DB에 저장되고 Web UI에 표시됨

### 파싱(Parsing)이란?

**파싱 = Python 파일을 읽어서 "이해"하는 과정**

Scheduler가 DAG 파일을 열어서:
1. 어떤 DAG이 있는지 찾고
2. 그 안에 Task가 뭐가 있는지 파악하고
3. Task 간 순서(의존성)가 어떻게 되는지 이해하고
4. 스케줄이 뭔지 확인

> 💡 Scheduler는 이 파싱을 **주기적으로 계속함** (기본 30초마다). 그래서 DAG 파일 수정하면 자동으로 반영됨

---

## 4. 실제 코드 예시

### 기본 예시 (Task 1개)

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

# DAG 정의 (작업 순서도)
with DAG(
    dag_id='my_first_dag',
    start_date=datetime(2024, 1, 1),
    schedule='@daily'
) as dag:

    # Task 정의
    def say_hello():
        print('Hello, Airflow!')

    hello_task = PythonOperator(
        task_id='say_hello',
        python_callable=say_hello
    )
```

### ETL 예시 (Task 3개 + 의존성)

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

# DAG 정의 (작업 순서도)
with DAG(
    dag_id='my_first_dag',
    start_date=datetime(2024, 1, 1),
    schedule='@daily'
) as dag:

    # Task 1: 데이터 수집
    def extract_data():
        print('1단계: 데이터를 수집합니다!')
        return {'raw_data': [1, 2, 3, 4, 5]}

    # Task 2: 데이터 정제
    def transform_data():
        print('2단계: 데이터를 정제합니다!')
        return {'clean_data': [2, 4, 6, 8, 10]}

    # Task 3: 결과 저장
    def load_data():
        print('3단계: 결과를 저장합니다!')
        print('완료!')

    # Task 생성
    task_extract = PythonOperator(
        task_id='extract',
        python_callable=extract_data
    )

    task_transform = PythonOperator(
        task_id='transform',
        python_callable=transform_data
    )

    task_load = PythonOperator(
        task_id='load',
        python_callable=load_data
    )

    # 의존성 설정 (실행 순서)
    # extract → transform → load
    task_extract >> task_transform >> task_load
```

### 의존성 설정 방법

```python
# 방법 1: >> 연산자 (가장 많이 씀)
task_a >> task_b >> task_c

# 방법 2: set_downstream
task_a.set_downstream(task_b)

# 방법 3: set_upstream (반대 방향)
task_b.set_upstream(task_a)

# 병렬 실행 후 합치기
#     → task_b →
# task_a          → task_d
#     → task_c →
task_a >> [task_b, task_c] >> task_d
```

---

## 5. 왜 Airflow를 쓸까?

### Airflow가 없을 때

- 크론탭(crontab)으로 스케줄링 → 작업 간 의존성 관리 어려움
- 실패 시 알림 → 직접 구현해야 함
- 재실행 → 수동으로 해야 함
- 모니터링 → 로그 파일 뒤져야 함

### Airflow가 있을 때

| 기능 | 장점 |
|------|------|
| 의존성 관리 | Task 간 순서를 코드로 명확하게 정의 |
| 자동 재시도 | 실패해도 자동으로 재시도 가능 |
| Web UI | 대시보드로 실시간 모니터링 |
| 알림 | Slack, Email 등으로 알림 설정 가능 |
| 백필(Backfill) | 과거 날짜 데이터도 쉽게 재처리 |

### 언제 쓰면 좋을까?

- 정해진 시간에 자동으로 실행해야 하는 배치 작업이 있을 때
- 여러 작업이 순서대로 실행되어야 할 때 (A → B → C)
- ETL (Extract-Transform-Load) 파이프라인을 구축할 때
- ML 모델 학습/배포를 자동화할 때
- 작업 실패 시 알림이나 자동 재시도가 필요할 때

---

## 6. Operator 심화

### 기본 제공 Operator (Built-in)

Airflow 설치하면 바로 쓸 수 있는 것들:

| Operator | import 경로 |
|----------|-------------|
| BashOperator | `airflow.operators.bash` |
| PythonOperator | `airflow.operators.python` |
| EmailOperator | `airflow.operators.email` |
| DummyOperator | `airflow.operators.dummy` (테스트용) |

### Provider 패키지 (추가 설치 필요)

SQL 관련 Operator는 DB 종류에 따라 별도 패키지를 설치해야 해요:

```bash
# MySQL 쓸 때
pip install apache-airflow-providers-mysql

# PostgreSQL 쓸 때
pip install apache-airflow-providers-postgres

# Snowflake 쓸 때
pip install apache-airflow-providers-snowflake
```

### 주요 Provider 패키지

| Provider | 용도 |
|----------|------|
| `apache-airflow-providers-google` | GCP, BigQuery, GCS |
| `apache-airflow-providers-amazon` | AWS, S3, Redshift |
| `apache-airflow-providers-slack` | Slack 알림 |
| `apache-airflow-providers-http` | REST API 호출 |

### Custom Operator 만들기

회사 내부 시스템 연동이나 반복되는 로직을 재사용할 때 유용합니다.

```python
from airflow.models import BaseOperator

class MyCustomOperator(BaseOperator):
    
    def __init__(self, my_param, **kwargs):
        super().__init__(**kwargs)
        self.my_param = my_param
    
    def execute(self, context):
        # 여기에 실제 로직 작성
        print(f"내 파라미터: {self.my_param}")
        return "작업 완료!"
```

> 💡 **팁:** 실무에서는 간단한 작업은 `PythonOperator`에 함수 넣어서 쓰고, 정말 여러 DAG에서 반복적으로 쓰이는 로직만 Custom Operator로 만드는 경우가 많아요.

---

## 7. XCom - Task 간 데이터 전달

### XCom이란?

**XCom = Task 간에 데이터를 주고받는 방법**

X(cross) + Com(communication) = **Task 간 통신**

### 왜 필요할까?

각 Task는 **독립적으로 실행**되기 때문에 그냥은 데이터를 넘길 수 없음

```python
def extract_data():
    return {'raw_data': [1, 2, 3, 4, 5]}  # 이 데이터를...

def transform_data():
    # 여기서 어떻게 받지?? 🤔
```

### XCom 사용법

```python
# Task 1: 데이터 보내기 (push)
def extract_data(**context):
    data = [1, 2, 3, 4, 5]
    context['ti'].xcom_push(key='raw_data', value=data)
    # 또는 그냥 return하면 자동으로 push됨
    return data

# Task 2: 데이터 받기 (pull)
def transform_data(**context):
    raw_data = context['ti'].xcom_pull(task_ids='extract', key='return_value')
    print(f'받은 데이터: {raw_data}')
    cleaned = [x * 2 for x in raw_data]
    return cleaned
```

### 핵심 정리

| 용어 | 의미 |
|------|------|
| `xcom_push` | 데이터 보내기 |
| `xcom_pull` | 데이터 가져오기 |
| `return` | 자동으로 push됨 |

### ⚠️ 주의: DataFrame 통째로 넘기면 안 되는 이유

XCom으로 데이터를 넘기면 **Metadata DB에 직접 저장**됩니다!

```
Task A → return df → [DB에 저장] → Task B가 pull → [DB에서 읽음]
```

| 문제 | 설명 |
|------|------|
| DB 용량 폭발 | DataFrame이 100MB라면? 매일 실행하면? DB 금방 터진대요 |
| 속도 저하 | 큰 데이터를 DB에 쓰고 읽고... 느려요 |
| 직렬화 이슈 | DataFrame을 DB에 저장하려면 변환(직렬화)이 필요한데, 이것도 비용 |

### 올바른 방법

```python
# ❌ 이렇게 하면 안 됨
def extract():
    df = pd.read_csv('huge_file.csv')  # 500MB짜리
    return df  # DB에 500MB 저장... 돈두댓

# ✅ 이렇게 해야 함
def extract():
    df = pd.read_csv('huge_file.csv')
    path = 's3://my-bucket/data/output.parquet'
    df.to_parquet(path)
    return path  # 경로만 넘김 (몇 글자)

def transform(**context):
    path = context['ti'].xcom_pull(task_ids='extract')
    df = pd.read_parquet(path)  # S3에서 직접 읽음
```

---

## 참고 자료

- [Apache Airflow 공식 문서](https://airflow.apache.org/docs/)
- [Airflow Provider 패키지 목록](https://airflow.apache.org/docs/apache-airflow-providers/)
