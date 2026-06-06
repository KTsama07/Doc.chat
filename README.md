# Doc.Chat

This repository contains `Doc_Chat.ipynb`, a Colab-first prototype for querying a road-safety CSV dataset with natural language using Gemini, DuckDB, FastAPI, and a web UI.

## 1. How to deploy

### Prerequisites
- Google Colab runtime
- Gemini API key (`GEMINI_API_KEY`)
- ngrok auth token (`NGROK_AUTH_TOKEN`)

### Deployment steps
1. Open `Doc_Chat.ipynb` from this repository in Google Colab.
2. Add both secrets in Colab **Secrets**:
   - `GEMINI_API_KEY`
   - `NGROK_AUTH_TOKEN`
3. Run notebook cells in order:
   - install dependencies
   - load/download dataset
   - initialize schema, SQL engine, orchestration, API, and frontend
4. Run the **Launch** cell to start Uvicorn and create an ngrok tunnel.
5. Open the printed **Frontend URL** to use Doc.Chat, and `/docs` for FastAPI docs.

---

## 2. Architecture diagram

```mermaid
flowchart LR
    U[User Browser UI] -->|POST /api/query| API[FastAPI Service]
    U -->|GET /api/audit| API
    U -->|GET /api/dataset/info| API

    API --> ORCH[Orchestrator]
    ORCH --> LLM[Gemini Structured LLM]
    ORCH --> SQLV[SQL Validator]
    SQLV --> DDB[DuckDB on CSV Dataset]
    ORCH --> SUM[Gemini Summary LLM]
    ORCH --> CHART[Plotly Chart Engine]
    ORCH --> AUDIT[(SQLite Audit DB)]

    API -->|JSON + chart payload| U
```

---

## 3. Query data flow and control flow diagram

```mermaid
sequenceDiagram
    participant User as User
    participant UI as Frontend
    participant API as FastAPI /api/query
    participant O as orchestrate()
    participant LLM as Gemini (structured)
    participant SQL as SQL Validator + DuckDB
    participant SUM as Gemini (summary)
    participant A as Audit DB

    User->>UI: Enter question
    UI->>API: POST question
    API->>O: orchestrate(question)
    O->>LLM: Build prompt + request QueryParameters
    LLM-->>O: is_in_scope, confidence, sql_query, chart metadata

    alt out of scope
        O->>A: log REFUSED
        O-->>API: refusal response
    else low confidence
        O->>A: log FLAGGED
        O-->>API: human-intervention message + SQL
    else valid
        O->>SQL: validate_sql(sql_query)
        SQL->>SQL: execute_query() on CSV
        SQL-->>O: dataframe
        O->>SUM: generate summary from dataframe
        SUM-->>O: natural-language answer
        O->>A: log SUCCESS
        O-->>API: answer + sql + chart + table data
    end

    API-->>UI: JSON response
    UI-->>User: Answer, SQL, chart, table
```
