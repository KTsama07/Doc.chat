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

Reference HLD:

![Doc.Chat HLD](https://github.com/user-attachments/assets/f7d5ded1-ae94-4332-99ec-5f4adc1f23c9)

```mermaid
flowchart TB
    U((User)) -->|1. Plain English Question| UI[Gradio UI Interface]
    UI -->|2. Forward Request| ORCH[Python Orchestrator]
    ORCH -->|3. Intent Routing| SCOPE{LLM: Scope & Intent Check}
    SCOPE -->|4a. Out of Scope| REF[Graceful Refusal State]
    REF -->|Return Text| UI
    SCOPE -->|4b. In Scope| EXTRACT[LLM: Extract to JSON Schema]
    EXTRACT -->|5. JSON Payload| PARSE[Backend: Parse JSON to SQL]
    PARSE -->|6. SQL Query| DDB[(DuckDB Engine)]
    DDB -->|8. Execute SELECT| CSV[(MORTH CSV Dataset)]
    DDB -->|9. Raw Result Set| ORCH
    ORCH -->|10. Feed Raw Data| SUM[LLM: Natural Language Summary]
    ORCH -->|11. Feed Raw Data| PLOT[Plotly: Generate Chart]
    PARSE -.->|7. Log SQL String| AUDIT[Audit Trail / Provenance]
    SUM --> RESP[Response Assembly]
    PLOT --> RESP
    AUDIT -.->|Append Provenance| RESP
    RESP -->|12. Final Payload| UI
```

---

## 3. Query data flow and control flow diagram

```mermaid
sequenceDiagram
    participant User as User
    participant UI as Gradio UI Interface
    participant O as Python Orchestrator
    participant IC as LLM Scope/Intent Check
    participant JE as LLM JSON Extractor
    participant B as Backend SQL Parser
    participant DB as DuckDB + MORTH CSV
    participant S as LLM Summary
    participant P as Plotly
    participant A as Audit Trail
    participant R as Response Assembly

    User->>UI: 1. Plain English Question
    UI->>O: 2. Forward Request
    O->>IC: 3. Intent Routing

    alt 4a. Out of Scope
        IC-->>UI: Graceful refusal text
    else 4b. In Scope
        IC->>JE: Route in-scope query
        JE->>B: 5. JSON Payload
        B->>DB: 6. SQL Query
        B-->>A: 7. Log SQL String
        DB->>DB: 8. Execute SELECT
        DB-->>O: 9. Raw Result Set
        O->>S: 10. Feed Raw Data
        O->>P: 11. Feed Raw Data
        S-->>R: Summary output
        P-->>R: Chart output
        A-->>R: Append provenance
        R-->>UI: 12. Final Payload
    end

    UI-->>User: Return response text + chart + provenance
```
