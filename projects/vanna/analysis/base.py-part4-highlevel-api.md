# base.py (Part 4) - 고수준 API (ask & train)

> **파일**: `src/vanna/base/base.py` (line 1683-2118)
> **역할**: 사용자가 직접 호출하는 메인 API - 전체 워크플로우 통합
> **주요 메서드**: `ask()`, `train()`, `get_training_plan_*()`, `get_plotly_figure()`

---

## 📋 파일 개요

### 기본 정보
- **경로**: `src/vanna/base/base.py`
- **분석 범위**: line 1683-2118 (약 435 lines)
- **주요 클래스**: `VannaBase` (고수준 API)
- **핵심 역할**: RAG + SQL 생성 + 실행 + 시각화를 하나의 메서드로 통합

### 핵심 역할 (한 문장)
**"`ask()`는 질문만 받아서 SQL 생성→실행→시각화→자동 학습까지 한 번에 처리하고, `train()`은 DDL/SQL/문서를 벡터 DB에 추가하는 사용자 친화적 API"**

### 누가 사용하는가?
- **일반 사용자**: `vn.ask("top customers")` 한 줄로 모든 것 해결
- **데이터 팀**: `vn.train(ddl=...)` 으로 스키마 학습
- **Jupyter 사용자**: 대화형으로 질문 → 결과 → 시각화

---

## 🔍 해결하는 핵심 문제들

### 문제 1: 사용자 경험 - 복잡한 워크플로우를 하나로

**문제**
- 실제 워크플로우는 복잡함:
  1. SQL 생성 (`generate_sql`)
  2. SQL 유효성 검증 (`is_sql_valid`)
  3. SQL 실행 (`run_sql`)
  4. 차트 생성 (`generate_plotly_code`)
  5. 차트 렌더링 (`get_plotly_figure`)
  6. 자동 학습 (`add_question_sql`)
- 사용자가 이 6단계를 매번 호출해야 한다면?

**문제가 없었다면?**
```python
# 사용자가 매번 이렇게 해야 함 ❌
sql = vn.generate_sql("top customers")
if vn.is_sql_valid(sql):
    df = vn.run_sql(sql)
    if vn.should_generate_chart(df):
        plotly_code = vn.generate_plotly_code(sql=sql, df_metadata=...)
        fig = vn.get_plotly_figure(plotly_code, df)
        fig.show()
    vn.add_question_sql("top customers", sql)  # 학습
```

**고민했던 선택지**

**선택지 1: 각 단계를 별도 메서드로 제공**
```python
# 사용자가 직접 조립
vn.generate_sql(...)
vn.run_sql(...)
vn.visualize(...)
```
- ✅ 장점: 유연함, 각 단계 제어 가능
- ❌ 단점: 사용자 부담, 실수 가능성
- 왜 안 됨: 대부분 사용자는 전체 워크플로우 필요

**선택지 2 (최종): `ask()` 하나로 모든 것 처리**
```python
# 한 줄로 끝!
sql, df, fig = vn.ask("Show me top 10 customers by sales")
```
- ✅ 장점:
  - 초보자 친화적
  - 에러 처리 내장
  - 옵션으로 커스터마이징 가능
- ⚠️ 단점: 내부 동작 숨겨짐
- 왜 선택: 사용자 경험 최우선

**최종 해결책**

```python
def ask(
    self,
    question: Union[str, None] = None,
    print_results: bool = True,
    auto_train: bool = True,
    visualize: bool = True,
    allow_llm_to_see_data: bool = False,
) -> Union[Tuple[str, pd.DataFrame, plotly.graph_objs.Figure], None]:
    """
    한 번의 호출로 전체 워크플로우 실행:
    1. SQL 생성
    2. SQL 실행
    3. 결과 출력
    4. 시각화 생성 & 표시
    5. 자동 학습 (옵션)
    """
    # 0. 질문 입력 (interactive)
    if question is None:
        question = input("Enter a question: ")

    # 1. SQL 생성
    try:
        sql = self.generate_sql(question=question, allow_llm_to_see_data=allow_llm_to_see_data)
    except Exception as e:
        print(e)
        return None, None, None

    # 2. SQL 출력 (Jupyter에서 syntax highlighting)
    if print_results:
        try:
            Code = __import__("IPython.display", fromList=["Code"]).Code
            display(Code(sql))
        except:
            print(sql)

    # 3. DB 연결 체크
    if self.run_sql_is_set is False:
        print("If you want to run the SQL query, connect to a database first.")
        return sql, None, None if not print_results else None

    # 4. SQL 실행
    try:
        df = self.run_sql(sql)

        # 결과 출력
        if print_results:
            try:
                display = __import__("IPython.display", fromList=["display"]).display
                display(df)
            except:
                print(df)

        # 5. 자동 학습 (결과가 있고, auto_train=True일 때)
        if len(df) > 0 and auto_train:
            self.add_question_sql(question=question, sql=sql)

        # 6. 시각화 (visualize=True일 때)
        if visualize:
            try:
                plotly_code = self.generate_plotly_code(
                    question=question,
                    sql=sql,
                    df_metadata=f"Running df.dtypes gives:\n {df.dtypes}"
                )
                fig = self.get_plotly_figure(plotly_code=plotly_code, df=df)

                # 차트 표시
                if print_results:
                    try:
                        Image = __import__("IPython.display", fromlist=["Image"]).Image
                        img_bytes = fig.to_image(format="png", scale=2)
                        display(Image(img_bytes))
                    except:
                        fig.show()

                return sql, df, fig
            except Exception as e:
                print("Couldn't run plotly code: ", e)
                return sql, df, None
        else:
            return sql, df, None

    except Exception as e:
        print("Couldn't run sql: ", e)
        return sql, None, None
```

**핵심 아이디어**
1. **Pipeline Pattern**: 여러 단계를 하나로 연결
2. **Graceful Degradation**: 중간 단계 실패해도 계속 진행
3. **Flexible Output**: `print_results` 플래그로 interactive/programmatic 모두 지원
4. **Auto-training Loop**: 성공한 쿼리는 자동 학습 → 점점 정확해짐

**트레이드오프**
- 얻은 것: 최고의 UX, 초보자 친화적
- 희생한 것: 내부 동작 가시성 (디버깅 어려울 수 있음)

---

### 문제 2: 학습 데이터 추가 - 통일된 인터페이스

**문제**
- 사용자가 추가할 수 있는 데이터 3가지:
  1. DDL (스키마)
  2. Question-SQL 쌍 (예시)
  3. Documentation (비즈니스 용어)
- 각각 별도 메서드 vs 하나의 메서드?

**문제가 없었다면?**
```python
# 사용자가 구분해서 호출
vn.add_ddl("CREATE TABLE ...")
vn.add_question_sql("top customers", "SELECT ...")
vn.add_documentation("VIP = sales > 10000")
```
- ✅ 명확함
- ❌ 사용자가 타입 구분해야 함

**고민했던 선택지**

**선택지 1: 별도 메서드만 제공**
```python
vn.add_ddl(...)
vn.add_question_sql(...)
vn.add_documentation(...)
```
- ✅ 장점: 명확, 타입 안전
- ❌ 단점: 사용자가 타입 판단

**선택지 2 (최종): `train()` 하나로 통합 + 자동 감지**
```python
# DDL
vn.train(ddl="CREATE TABLE customers (...)")

# SQL (자동으로 질문 생성)
vn.train(sql="SELECT * FROM customers ORDER BY sales DESC LIMIT 10")
# → LLM이 질문 자동 생성: "What are the top 10 customers by sales?"

# Question + SQL
vn.train(
    question="Show top customers",
    sql="SELECT * FROM customers ORDER BY sales DESC LIMIT 10"
)

# Documentation
vn.train(documentation="VIP customers = sales > 10000")

# Training Plan (일괄)
plan = vn.get_training_plan_snowflake()
vn.train(plan=plan)
```
- ✅ 장점:
  - 하나의 인터페이스
  - SQL만 있어도 질문 자동 생성
  - Training Plan으로 일괄 처리
- ⚠️ 단점: 내부에서 타입 분기
- 왜 선택: 사용자 편의성

**최종 해결책**

```python
def train(
    self,
    question: str = None,
    sql: str = None,
    ddl: str = None,
    documentation: str = None,
    plan: TrainingPlan = None,
) -> str:
    """
    통합 학습 인터페이스
    - question + sql: Question-SQL 쌍 추가
    - sql만: LLM이 질문 자동 생성
    - ddl: DDL 추가
    - documentation: 문서 추가
    - plan: Training Plan 일괄 처리
    """
    # 검증: question만 있으면 에러
    if question and not sql:
        raise ValidationError("Please also provide a SQL query")

    # 1. Documentation
    if documentation:
        print("Adding documentation....")
        return self.add_documentation(documentation)

    # 2. SQL
    if sql:
        if question is None:
            # SQL만 있으면 LLM이 질문 생성
            question = self.generate_question(sql)
            print("Question generated with sql:", question, "\nAdding SQL...")
        return self.add_question_sql(question=question, sql=sql)

    # 3. DDL
    if ddl:
        print("Adding ddl:", ddl)
        return self.add_ddl(ddl)

    # 4. Training Plan
    if plan:
        for item in plan._plan:
            if item.item_type == TrainingPlanItem.ITEM_TYPE_DDL:
                self.add_ddl(item.item_value)
            elif item.item_type == TrainingPlanItem.ITEM_TYPE_IS:
                self.add_documentation(item.item_value)
            elif item.item_type == TrainingPlanItem.ITEM_TYPE_SQL:
                self.add_question_sql(question=item.item_name, sql=item.item_value)
```

**핵심 아이디어**
1. **Smart Defaults**: SQL만 있어도 질문 자동 생성
2. **Bulk Training**: Training Plan으로 수백 개 일괄 추가
3. **Type Dispatch**: 파라미터로 타입 자동 판단

**트레이드오프**
- 얻은 것: 통일된 인터페이스, 편리함
- 희생한 것: 내부 타입 분기 로직

---

### 문제 3: Training Plan - DB 메타데이터 자동 학습

**문제**
- Snowflake에 100개 테이블
- 각 테이블마다 DDL을 수동으로 `train(ddl=...)`?
- 너무 번거로움!

**문제가 없었다면?**
```python
# 수동으로 100개 테이블 학습 ❌
for table in get_all_tables():
    ddl = get_ddl(table)
    vn.train(ddl=ddl)
```

**고민했던 선택지**

**선택지 1: 사용자가 직접 루프**
```python
for table in tables:
    vn.train(ddl=get_ddl(table))
```
- ✅ 장점: 간단
- ❌ 단점: 사용자 부담
- 왜 안 됨: Vanna가 DB 메타데이터 접근 가능한데 왜?

**선택지 2 (최종): Training Plan 자동 생성**
```python
# Snowflake 메타데이터 자동 수집
plan = vn.get_training_plan_snowflake(
    filter_databases=["production"],
    filter_schemas=["public"],
    use_historical_queries=True  # 과거 쿼리도 학습!
)

# 일괄 학습
vn.train(plan=plan)
```
- ✅ 장점:
  - 자동화 (INFORMATION_SCHEMA 활용)
  - 과거 쿼리 히스토리도 학습
  - 필터링 지원
- ⚠️ 단점: Snowflake 전용 (DB마다 구현 필요)
- 왜 선택: 사용자 경험 극대화

**최종 해결책**

```python
def get_training_plan_snowflake(
    self,
    filter_databases: List[str] = None,
    filter_schemas: List[str] = None,
    include_information_schema: bool = False,
    use_historical_queries: bool = True,
) -> TrainingPlan:
    """
    Snowflake 메타데이터를 자동으로 수집하여 Training Plan 생성
    """
    plan = TrainingPlan([])

    # 1. 과거 쿼리 히스토리 (옵션)
    if use_historical_queries:
        try:
            df_history = self.run_sql(
                """
                SELECT * FROM table(information_schema.query_history(result_limit => 5000))
                ORDER BY start_time
                """
            )

            # 필터링 (결과 있는 쿼리만)
            df_history_filtered = df_history.query("ROWS_PRODUCED > 1")

            # 데이터베이스 필터
            if filter_databases:
                mask = df_history_filtered["QUERY_TEXT"].str.lower().apply(
                    lambda x: any(s in x for s in [s.lower() for s in filter_databases])
                )
                df_history_filtered = df_history_filtered[mask]

            # 샘플링 (너무 많으면)
            if len(df_history_filtered) > 10:
                df_history_filtered = df_history_filtered.sample(10)

            # Training Plan에 추가
            for query in df_history_filtered["QUERY_TEXT"].unique().tolist():
                plan._plan.append(
                    TrainingPlanItem(
                        item_type=TrainingPlanItem.ITEM_TYPE_SQL,
                        item_group="",
                        item_name=self.generate_question(query),  # LLM이 질문 생성
                        item_value=query
                    )
                )
        except Exception as e:
            print(f"Could not get query history: {e}")

    # 2. 데이터베이스 메타데이터
    databases = self._get_databases()

    for database in databases:
        if filter_databases and database not in filter_databases:
            continue

        # INFORMATION_SCHEMA.COLUMNS 조회
        df_columns = self.run_sql(
            f"SELECT * FROM {database}.INFORMATION_SCHEMA.COLUMNS"
        )

        for schema in df_columns["TABLE_SCHEMA"].unique():
            if filter_schemas and schema not in filter_schemas:
                continue

            if not include_information_schema and schema == "INFORMATION_SCHEMA":
                continue

            # 테이블별로 컬럼 정보 묶기
            for table in df_columns[df_columns["TABLE_SCHEMA"] == schema]["TABLE_NAME"].unique():
                df_table = df_columns[
                    (df_columns["TABLE_SCHEMA"] == schema) &
                    (df_columns["TABLE_NAME"] == table)
                ]

                # Markdown 테이블로 변환
                doc = f"The following columns are in the {table} table in the {database} database:\n\n"
                doc += df_table[["TABLE_CATALOG", "TABLE_SCHEMA", "TABLE_NAME", "COLUMN_NAME", "DATA_TYPE", "COMMENT"]].to_markdown()

                plan._plan.append(
                    TrainingPlanItem(
                        item_type=TrainingPlanItem.ITEM_TYPE_IS,  # Information Schema
                        item_group=f"{database}.{schema}",
                        item_name=table,
                        item_value=doc
                    )
                )

    return plan
```

**핵심 아이디어**
1. **Metadata Mining**: `INFORMATION_SCHEMA` 활용
2. **Query History**: 과거 쿼리에서 패턴 학습
3. **Intelligent Sampling**: 너무 많으면 샘플링 (10개)
4. **Markdown Documentation**: 테이블 정보를 LLM이 이해하기 쉬운 형식으로

**트레이드오프**
- 얻은 것: 자동화, 대규모 DB도 쉽게 학습
- 희생한 것: DB별 구현 필요 (Snowflake, generic만 있음)

---

### 문제 4: 시각화 - Plotly 코드 실행 안전성

**문제**
- LLM이 생성한 Plotly 코드를 `exec()` 실행
- 악의적 코드 가능성? (`os.system("rm -rf /")`)
- 잘못된 코드로 크래시?

**문제가 없었다면?**
```python
# LLM 생성 코드 그대로 실행 ❌
plotly_code = llm.generate(...)
exec(plotly_code)  # 위험!
```

**고민했던 선택지**

**선택지 1: Sandbox 실행 환경**
```python
import subprocess
subprocess.run(["python", "-c", plotly_code], timeout=5)
```
- ✅ 장점: 완전 격리
- ❌ 단점: 복잡, 느림, DataFrame 전달 어려움
- 왜 안 됨: 오버킬

**선택지 2 (최종): 제한된 Namespace + 에러 처리 + Fallback**
```python
def get_plotly_figure(
    self, plotly_code: str, df: pd.DataFrame, dark_mode: bool = True
) -> plotly.graph_objs.Figure:
    # 제한된 namespace (df, px, go만)
    ldict = {"df": df, "px": px, "go": go}

    try:
        # LLM 코드 실행
        exec(plotly_code, globals(), ldict)
        fig = ldict.get("fig", None)
    except Exception as e:
        # 실패 시 자동 Fallback 차트 생성
        numeric_cols = df.select_dtypes(include=["number"]).columns.tolist()
        categorical_cols = df.select_dtypes(include=["object", "category"]).columns.tolist()

        # 휴리스틱 기반 차트 선택
        if len(numeric_cols) >= 2:
            fig = px.scatter(df, x=numeric_cols[0], y=numeric_cols[1])
        elif len(numeric_cols) == 1 and len(categorical_cols) >= 1:
            fig = px.bar(df, x=categorical_cols[0], y=numeric_cols[0])
        elif len(categorical_cols) >= 1 and df[categorical_cols[0]].nunique() < 10:
            fig = px.pie(df, names=categorical_cols[0])
        else:
            fig = px.line(df)

    if fig is None:
        return None

    # Dark mode 적용
    if dark_mode:
        fig.update_layout(template="plotly_dark")

    return fig
```
- ✅ 장점:
  - 제한된 namespace (보안)
  - 에러 시 자동 Fallback (안정성)
  - 사용자는 항상 차트 볼 수 있음
- ⚠️ 단점: 완벽한 보안은 아님 (eval 위험성 여전히 존재)
- 왜 선택: 실용성 + 안정성 균형

**핵심 아이디어**
1. **Limited Globals**: `exec(..., globals(), ldict)` - 제한된 변수만
2. **Graceful Fallback**: LLM 실패 시 휴리스틱 차트
3. **Always Show Something**: 사용자는 항상 시각화 받음

**트레이드오프**
- 얻은 것: 안정성, 항상 차트 생성
- 희생한 것: 완벽한 보안은 아님 (신뢰할 수 있는 LLM만 사용)

---

### 문제 5: Interactive vs Programmatic - 두 가지 사용 패턴

**문제**
- Jupyter: 대화형, 결과 즉시 출력
- Production: 프로그래매틱, 결과 반환만

**고민했던 선택지**

**선택지 1: 별도 메서드**
```python
vn.ask_interactive(...)  # Jupyter
vn.ask_programmatic(...)  # Production
```
- ✅ 명확함
- ❌ API 중복
- 왜 안 됨: 하나로 충분

**선택지 2 (최종): `print_results` 플래그**
```python
# Jupyter (interactive)
vn.ask("top customers")  # print_results=True (기본)
# → SQL, DataFrame, Figure 모두 출력

# Production (programmatic)
sql, df, fig = vn.ask("top customers", print_results=False)
# → 반환만, 출력 없음
```
- ✅ 하나의 API로 두 모드 지원
- ⚠️ 플래그 이해 필요
- 왜 선택: API 단순성

**핵심 아이디어**
1. **Mode Flag**: `print_results`로 출력 제어
2. **IPython Detection**: `__import__("IPython.display")` 시도
3. **Consistent Return**: 항상 튜플 반환

---

## ⭐ 실전 적용 가이드

### 가이드 1: 커스텀 워크플로우 구현

**상황**: `ask()`를 확장하여 슬랙 알림 추가

#### Step 1: 상속 & 오버라이드

```python
from vanna.base import VannaBase
import requests

class SlackVanna(VannaBase):
    def __init__(self, config):
        super().__init__(config)
        self.slack_webhook = config.get('slack_webhook')

    def ask(self, question, **kwargs):
        # 기존 ask 호출
        sql, df, fig = super().ask(question, print_results=False, **kwargs)

        # 슬랙 알림 추가
        if sql and df is not None:
            self.send_slack_notification(
                question=question,
                sql=sql,
                row_count=len(df)
            )

        return sql, df, fig

    def send_slack_notification(self, question, sql, row_count):
        message = {
            "text": f"Query executed: {question}",
            "attachments": [{
                "color": "good",
                "fields": [
                    {"title": "SQL", "value": f"```{sql}```", "short": False},
                    {"title": "Rows", "value": str(row_count), "short": True}
                ]
            }]
        }
        requests.post(self.slack_webhook, json=message)

# 사용
vn = SlackVanna(config={
    'api_key': '...',
    'slack_webhook': 'https://hooks.slack.com/...'
})

vn.ask("Show top customers")  # SQL 실행 & 슬랙 알림
```

---

### 가이드 2: Streaming 응답 구현

**상황**: LLM 응답을 실시간으로 보여주기

#### Step 1: 구현

```python
class StreamingVanna(VannaBase):
    def ask_streaming(self, question):
        print(f"Question: {question}\n")
        print("Generating SQL", end="")

        # SQL 생성 (progress)
        for i in range(3):
            print(".", end="", flush=True)
            time.sleep(0.5)

        sql = self.generate_sql(question)
        print(f"\n\n{sql}\n")

        # 실행
        print("Executing query", end="")
        df = self.run_sql(sql)
        print(f"✓ ({len(df)} rows)\n")

        # 시각화
        if len(df) > 0:
            print("Generating chart", end="")
            plotly_code = self.generate_plotly_code(
                question=question, sql=sql, df_metadata=str(df.dtypes)
            )
            fig = self.get_plotly_figure(plotly_code, df)
            print("✓\n")
            fig.show()

        return sql, df, fig
```

---

### 가이드 3: 배치 처리

**상황**: 100개 질문을 한 번에 처리

#### Step 1: 구현

```python
class BatchVanna(VannaBase):
    def ask_batch(self, questions: List[str], output_dir: str = "./results"):
        import os
        os.makedirs(output_dir, exist_ok=True)

        results = []
        for i, question in enumerate(questions):
            print(f"[{i+1}/{len(questions)}] {question}")

            try:
                sql, df, fig = self.ask(question, print_results=False, visualize=False)

                # 결과 저장
                df.to_csv(f"{output_dir}/result_{i}.csv", index=False)

                # 메타데이터 저장
                with open(f"{output_dir}/result_{i}.json", "w") as f:
                    json.dump({
                        "question": question,
                        "sql": sql,
                        "rows": len(df),
                        "success": True
                    }, f)

                results.append({"question": question, "success": True, "rows": len(df)})

            except Exception as e:
                print(f"  ❌ Error: {e}")
                results.append({"question": question, "success": False, "error": str(e)})

        # 요약
        success_rate = sum(1 for r in results if r['success']) / len(results)
        print(f"\n✅ Batch complete: {success_rate*100:.1f}% success rate")

        return results

# 사용
questions = [
    "Show top 10 customers",
    "What are the recent orders?",
    "Monthly sales trend",
    # ... 100개
]

vn = BatchVanna(config={...})
results = vn.ask_batch(questions)
```

---

## ⭐ 안티패턴과 흔한 실수

### 실수 1: `print_results=True`로 프로덕션 배포

**❌ 나쁜 예:**
```python
# Production FastAPI
@app.get("/query")
def query(question: str):
    vn.ask(question)  # ❌ print_results=True (기본값)
    # → 콘솔에 출력됨, 로그 폭발!
```

**문제:**
- 로그 파일 크기 폭발
- 출력 버퍼 block 가능성

**✅ 좋은 예:**
```python
@app.get("/query")
def query(question: str):
    sql, df, fig = vn.ask(question, print_results=False)
    return {
        "sql": sql,
        "data": df.to_dict(orient="records"),
        "chart": fig.to_json() if fig else None
    }
```

---

### 실수 2: `auto_train=True`를 항상 사용

**❌ 나쁜 예:**
```python
# 모든 쿼리를 자동 학습
vn.ask("random test query", auto_train=True)  # ❌
```

**문제:**
- 테스트 쿼리도 학습됨
- 나쁜 SQL도 학습됨
- Vector DB 오염

**✅ 좋은 예:**
```python
# 개발/테스트
sql, df, fig = vn.ask("test query", auto_train=False)

# 프로덕션 (사용자 확인 후)
sql, df, fig = vn.ask(question, auto_train=False, print_results=False)

# 사용자가 만족하면
if user_satisfied:
    vn.train(question=question, sql=sql)
```

---

### 실수 3: 에러 처리 없이 `ask()` 사용

**❌ 나쁜 예:**
```python
sql, df, fig = vn.ask(question)
print(df.head())  # ❌ df가 None일 수 있음!
```

**문제:**
- SQL 생성 실패 시 `df = None`
- `AttributeError: 'NoneType' object has no attribute 'head'`

**✅ 좋은 예:**
```python
sql, df, fig = vn.ask(question, print_results=False)

if df is None:
    print("Query failed or no results")
else:
    print(f"Got {len(df)} rows")
    print(df.head())
```

---

### 실수 4: Training Plan 없이 대규모 DB 학습

**❌ 나쁜 예:**
```python
# 100개 테이블을 수동으로
for table in all_tables:
    ddl = get_ddl(table)
    vn.train(ddl=ddl)  # ❌ 느리고 번거로움
```

**문제:**
- 수동 작업
- 과거 쿼리 히스토리 놓침

**✅ 좋은 예:**
```python
# Training Plan 사용
plan = vn.get_training_plan_snowflake(
    filter_databases=["production"],
    filter_schemas=["public"],
    use_historical_queries=True
)

print(f"Training plan has {len(plan._plan)} items")

# 일괄 학습
vn.train(plan=plan)
```

---

### 실수 5: `visualize=True`로 대규모 데이터 처리

**❌ 나쁜 예:**
```python
# 100만 행 결과
sql, df, fig = vn.ask(
    "Show all transactions",
    visualize=True  # ❌ Plotly가 100만 행 렌더링 시도
)
```

**문제:**
- 브라우저 크래시
- Plotly 코드 생성 실패

**✅ 좋은 예:**
```python
sql, df, fig = vn.ask(
    "Show all transactions",
    visualize=False  # ✅ 큰 데이터는 시각화 끄기
)

# 샘플링 후 시각화
if len(df) > 10000:
    df_sample = df.sample(1000)
    plotly_code = vn.generate_plotly_code(
        question="Sample of transactions",
        sql=sql,
        df_metadata=str(df_sample.dtypes)
    )
    fig = vn.get_plotly_figure(plotly_code, df_sample)
    fig.show()
```

---

### 실수 6: Plotly 코드 검증 없이 실행

**❌ 나쁜 예:**
```python
plotly_code = vn.generate_plotly_code(...)
exec(plotly_code)  # ❌ LLM이 잘못된 코드 생성 시 크래시
```

**문제:**
- `exec()` 에러 → 전체 워크플로우 중단

**✅ 좋은 예:**
```python
# Vanna의 get_plotly_figure가 이미 처리
fig = vn.get_plotly_figure(plotly_code, df)
# → 에러 시 자동 Fallback 차트

if fig is None:
    print("Could not generate chart")
else:
    fig.show()
```

---

### 실수 7: 질문 없이 `ask()` 호출 (non-interactive 환경)

**❌ 나쁜 예:**
```python
# Docker 컨테이너에서
vn.ask()  # ❌ input() 호출 → 무한 대기
```

**문제:**
- `question=None` → `input("Enter a question: ")` 호출
- Non-interactive 환경에서 block

**✅ 좋은 예:**
```python
# 항상 질문 명시
vn.ask("Show top customers")

# 또는 환경 체크
import sys
if sys.stdin.isatty():
    vn.ask()  # Interactive
else:
    raise ValueError("Question required in non-interactive mode")
```

---

## ⭐ 스케일 고려사항

### 소규모 (< 10 사용자, 간헐적 사용)

**권장 사항:**
- ✅ 기본 `ask()` 그대로 사용
- ✅ `auto_train=True` OK
- ✅ Jupyter Notebook 주 사용

**구현 예시:**
```python
vn = MyVanna(config={...})
vn.connect_to_postgres(...)

# 대화형으로 사용
vn.ask()  # Enter question 프롬프트
```

**모니터링:**
```python
# 기본 로깅만
import logging
logging.basicConfig(level=logging.INFO)
```

---

### 중규모 (10-100 사용자, 정기적 사용)

**권장 사항:**
- ✅ `print_results=False`로 API 서비스
- ✅ `auto_train=False` + 승인 워크플로우
- ✅ 결과 캐싱
- ✅ Training Plan 정기 업데이트

**구현 예시:**
```python
from functools import lru_cache
from fastapi import FastAPI

app = FastAPI()
vn = MyVanna(config={...})

@lru_cache(maxsize=100)
def cached_ask(question: str):
    return vn.ask(question, print_results=False, visualize=False)

@app.get("/query")
def query(question: str):
    sql, df, fig = cached_ask(question)
    return {
        "sql": sql,
        "data": df.to_dict(orient="records") if df is not None else None
    }

# Training Plan 정기 업데이트 (cron)
def update_training_plan():
    plan = vn.get_training_plan_snowflake(use_historical_queries=True)
    vn.train(plan=plan)

# 매일 밤 실행
```

**모니터링:**
```python
import time

class MonitoredVanna(MyVanna):
    def ask(self, question, **kwargs):
        start = time.time()
        try:
            result = super().ask(question, **kwargs)
            duration = time.time() - start
            print(f"[Query] {question[:50]}... took {duration:.2f}s")
            return result
        except Exception as e:
            print(f"[Error] {question[:50]}... failed: {e}")
            raise
```

---

### 대규모 (100+ 사용자, 실시간 서비스)

**권장 사항:**
- ✅ 비동기 처리 (Celery)
- ✅ Query Queue + Worker Pool
- ✅ 결과 캐싱 (Redis)
- ✅ Rate limiting
- ✅ Prometheus 메트릭

**구현 예시:**
```python
from celery import Celery
from prometheus_client import Counter, Histogram
import redis

# Celery 설정
celery_app = Celery('vanna', broker='redis://localhost:6379')

# Prometheus 메트릭
query_counter = Counter('vanna_queries_total', 'Total queries')
query_duration = Histogram('vanna_query_seconds', 'Query duration')

# Redis 캐시
cache = redis.Redis()

class ProductionVanna(MyVanna):
    def ask(self, question, **kwargs):
        query_counter.inc()

        # 캐시 확인
        cache_key = f"query:{hash(question)}"
        cached = cache.get(cache_key)
        if cached:
            return json.loads(cached)

        # 쿼리 실행
        with query_duration.time():
            sql, df, fig = super().ask(
                question,
                print_results=False,
                visualize=False,
                auto_train=False,
                **kwargs
            )

        # 캐시 저장 (1시간)
        result = {
            "sql": sql,
            "data": df.to_dict(orient="records") if df is not None else None
        }
        cache.setex(cache_key, 3600, json.dumps(result))

        return sql, df, fig

# Celery 태스크
@celery_app.task
def async_ask(question: str):
    vn = ProductionVanna(config={...})
    sql, df, fig = vn.ask(question)
    return {"sql": sql, "rows": len(df) if df is not None else 0}

# API
@app.post("/query")
def query(question: str):
    task = async_ask.delay(question)
    return {"task_id": task.id}

@app.get("/result/{task_id}")
def get_result(task_id: str):
    task = async_ask.AsyncResult(task_id)
    if task.ready():
        return {"status": "complete", "result": task.result}
    else:
        return {"status": "pending"}
```

**알림 설정:**
```yaml
# Prometheus alerts
- alert: HighQueryErrorRate
  expr: rate(vanna_query_errors_total[5m]) > 0.1
  annotations:
    summary: "Query error rate > 10%"

- alert: SlowQueries
  expr: histogram_quantile(0.95, vanna_query_seconds) > 30
  annotations:
    summary: "P95 query time > 30s"
```

---

## 💡 배운 점

### 1. 파이프라인 패턴으로 복잡도 숨기기
**핵심 개념**: 여러 단계를 하나의 API로 통합
**언제 사용?**: 사용자가 반복적으로 호출하는 워크플로우
**적용 가능한 곳**:
- ETL 파이프라인
- ML 예측 파이프라인

### 2. Smart Defaults로 UX 향상
**핵심 개념**: SQL만 주면 질문 자동 생성
**언제 사용?**: 사용자 부담 줄이고 싶을 때
**적용 가능한 곳**:
- 설정 파일 (defaults)
- API 파라미터

### 3. Graceful Degradation은 신뢰성
**핵심 개념**: Plotly 실패 → 자동 Fallback 차트
**언제 사용?**: 외부 의존성 (LLM) 사용 시
**적용 가능한 곳**:
- 외부 API 호출
- 추천 시스템

### 4. Mode Flag (print_results)로 다목적 API
**핵심 개념**: 하나의 API가 Jupyter + Production 모두 지원
**언제 사용?**: Interactive + Programmatic 모두 필요
**적용 가능한 곳**:
- CLI 도구 (verbose 플래그)
- 라이브러리 (debug 모드)

### 5. Training Plan은 자동화의 핵심
**핵심 개념**: 메타데이터 자동 수집 → 대규모 학습
**언제 사용?**: 반복 작업 자동화
**적용 가능한 곳**:
- DB 마이그레이션
- 설정 스캔

### 6. `exec()` 사용 시 Namespace 제한
**핵심 개념**: `exec(..., globals(), limited_locals)`
**언제 사용?**: 사용자 코드 실행 (플러그인, DSL)
**적용 가능한 곳**:
- 플러그인 시스템
- Configuration as Code

### 7. Auto-training Loop는 Self-improvement
**핵심 개념**: 성공한 쿼리 자동 학습 → 점점 정확해짐
**언제 사용?**: RAG, 추천 시스템
**적용 가능한 곳**:
- 검색 엔진 (클릭 피드백)
- 챗봇 (대화 학습)

### 8. Async + Queue는 대규모 서비스 필수
**핵심 개념**: 즉시 응답 + 백그라운드 처리
**언제 사용?**: 느린 작업 (LLM, DB 쿼리)
**적용 가능한 곳**:
- 이메일 발송
- 리포트 생성

---

## 📊 요약

| 항목 | 내용 |
|------|------|
| **핵심 문제** | 복잡한 RAG 워크플로우를 간단한 API로 |
| **핵심 패턴** | Pipeline + Smart Defaults + Graceful Degradation |
| **주요 트레이드오프** | 최고의 UX vs 내부 동작 가시성 |
| **핵심 기법** | 1) `ask()` 하나로 6단계 통합<br>2) `train()` 타입 자동 감지<br>3) Training Plan 자동 생성<br>4) Plotly Fallback 차트<br>5) Auto-training Loop |
| **적용 시 주의** | 1) `print_results=False` (프로덕션)<br>2) `auto_train=False` + 승인<br>3) None 체크 (df, fig)<br>4) Training Plan 사용<br>5) 대규모 데이터는 `visualize=False` |
| **스케일 전략** | 소규모: 기본 사용<br>중규모: API 서비스 + 캐싱<br>대규모: Async Queue + Redis + Prometheus |
| **실무 적용** | BI 도구, 데이터 대시보드, 자연어 쿼리 서비스 |

---

**✅ Part 4 (마지막) 분석 완료!**

---

## 🎉 base.py 전체 분석 완료!

**4개 Part 분석 완료:**
1. ✅ Part 1: RAG 워크플로우 (generate_sql, extract_sql)
2. ✅ Part 2: 추상 메서드 & Prompt 생성 (Mixin 패턴, 토큰 관리)
3. ✅ Part 3: DB 연결 패턴 (11개 DB, 동적 함수 바인딩)
4. ✅ Part 4: 고수준 API (ask, train, Training Plan)

**전체 요약:**
- **파일 크기**: 2118 lines
- **핵심 문제**: RAG 기반 Text-to-SQL 프레임워크
- **핵심 패턴**: Mixin + 동적 함수 바인딩 + Pipeline
- **실무 적용**: 사내 DB용 자연어 쿼리 인터페이스, BI 도구 대안
