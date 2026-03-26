# Architektura Systemu – Eskadra Bielik Misja 2

System RAG (Retrieval-Augmented Generation) oparty na polskim modelu językowym Bielik,
uruchomiony na Google Cloud Platform.

---

## Diagram Komponentów i Przepływów Informacji

```mermaid
flowchart TD
    User(["👤 Użytkownik\n(Przeglądarka)"])

    subgraph UI["🖥️ Web UI (index.html)"]
        direction TB
        UIForm["Formularz zapytania"]
        UIDual["Panel podwójny\nRAG vs Direct"]
        UICtx["Wyświetlanie\nkontekstu"]
    end

    subgraph ORCH["☁️ Cloud Run – Orchestration API (FastAPI)"]
        direction TB
        EP_ASK["POST /ask\n(RAG)"]
        EP_DIRECT["POST /ask_direct\n(Direct)"]
        EP_INGEST["POST /ingest\n(Ingestion)"]
        EP_ROOT["GET /\n(Serwowanie UI)"]

        subgraph RAG_FLOW["Ścieżka RAG Orkiestratora"]
            direction LR
            R1["1️⃣ Pobierz embedding\nzapytania"]
            R2["2️⃣ Wyszukaj wektory\n(BigQuery)"]
            R3["3️⃣ Zbuduj\naugmented prompt"]
            R4["4️⃣ Wyślij do LLM\n+ kontekst"]
            R1 --> R2 --> R3 --> R4
        end

        subgraph INGEST_FLOW["Ścieżka Ingestion"]
            direction LR
            I1["1️⃣ Parsuj CSV"]
            I2["2️⃣ Pobierz embedding\ndokumentu"]
            I3["3️⃣ Zapisz do\nBigQuery"]
            I1 --> I2 --> I3
        end

        subgraph DIRECT_FLOW["Ścieżka Direct"]
            direction LR
            D1["1️⃣ Wyślij zapytanie\nbezpośrednio do LLM"]
        end

        EP_ASK --> RAG_FLOW
        EP_INGEST --> INGEST_FLOW
        EP_DIRECT --> DIRECT_FLOW
    end

    subgraph EMBED["☁️ Cloud Run – Embedding Service\n(Ollama + EmbeddingGemma)\n8 CPU · 16 GB RAM"]
        EmbAPI["POST /api/embed\n→ wektor 768D (FLOAT64)"]
    end

    subgraph LLM["☁️ Cloud Run – LLM Service\n(Ollama + Bielik 4.5b)\n8 CPU · 32 GB RAM · GPU NVIDIA L4"]
        LLMApi["POST /api/chat\n→ odpowiedź w języku polskim"]
    end

    subgraph BQ["☁️ Google BigQuery\n(Vector Search)"]
        direction TB
        BQTable["Tabela: rag_dataset.hotel_rules\n──────────────────────────\nid       STRING  REQUIRED\ncontent  STRING  REQUIRED\nembedding FLOAT64 REPEATED (768D)"]
        BQSearch["VECTOR_SEARCH()\nCosineSimilarity · Top-K=3"]
        BQTable --> BQSearch
    end

    subgraph IAM["🔐 Google Cloud IAM"]
        Token["Identity Token\n(fetch_id_token)"]
    end

    %% ── User ↔ UI ──────────────────────────────────────────────────
    User -->|"Wpisuje pytanie"| UIForm
    UIForm -->|"Wysyła żądanie"| EP_ASK
    UIForm -->|"Wysyła żądanie"| EP_DIRECT
    UIForm -->|"Przesyła plik CSV"| EP_INGEST
    EP_ROOT -->|"Zwraca HTML"| UIDual
    EP_ASK  -->|"{ answer, context_used }"| UIDual
    EP_DIRECT -->|"{ answer }"| UIDual
    UIDual --> UICtx

    %% ── RAG Flow ───────────────────────────────────────────────────
    R1 -->|"POST /api/embed\n(tekst zapytania)"| EmbAPI
    EmbAPI -->|"wektor 768D"| R2
    R2 -->|"Zapytanie SQL\nVECTOR_SEARCH()"| BQSearch
    BQSearch -->|"Top 3 dokumenty"| R3
    R3 -->|"Augmented prompt\n[kontekst + pytanie]"| R4
    R4 -->|"POST /api/chat"| LLMApi

    %% ── Ingest Flow ────────────────────────────────────────────────
    I2 -->|"POST /api/embed\n(tekst dokumentu)"| EmbAPI
    EmbAPI -->|"wektor 768D"| I3
    I3 -->|"INSERT rows\n{id, content, embedding}"| BQTable

    %% ── Direct Flow ────────────────────────────────────────────────
    D1 -->|"POST /api/chat\n(surowe pytanie)"| LLMApi

    %% ── Auth ───────────────────────────────────────────────────────
    ORCH -->|"Żąda tokenu\ndostępu"| Token
    Token -->|"Bearer token\ndla prywatnych usług"| EMBED
    Token -->|"Bearer token\ndla prywatnych usług"| LLM
```

---

## Opis Komponentów

| Komponent | Technologia | Zasoby | Dostęp |
|---|---|---|---|
| **Web UI** | HTML5 + Vanilla JS + Marked.js | — | Publiczny |
| **Orchestration API** | FastAPI + Uvicorn (Python 3.11) | Cloud Run, maks. 2 instancje | Publiczny |
| **Embedding Service** | Ollama + EmbeddingGemma | 8 CPU, 16 GB RAM, maks. 1 instancja | Prywatny (IAM) |
| **LLM Service** | Ollama + Bielik 4.5b-v3.0-instruct Q8_0 | 8 CPU, 32 GB RAM, GPU NVIDIA L4, maks. 1 instancja | Prywatny (IAM) |
| **BigQuery Vector Store** | Google BigQuery + Vector Search | Zestaw danych: `rag_dataset`, tabela: `hotel_rules` | Prywatny (IAM) |

---

## Ścieżki Orkiestratora

### 1. Ścieżka RAG (POST /ask)

```mermaid
sequenceDiagram
    actor User as Użytkownik
    participant UI as Web UI
    participant API as Orchestration API
    participant Emb as Embedding Service
    participant BQ as BigQuery
    participant LLM as Bielik LLM

    User->>UI: Wpisuje pytanie
    UI->>API: POST /ask { "query": "..." }
    API->>Emb: POST /api/embed (tekst zapytania)
    Emb-->>API: wektor 768D

    API->>BQ: VECTOR_SEARCH(wektor, top_k=3)
    BQ-->>API: 3 najistotniejsze dokumenty

    API->>API: Buduje augmented prompt\n(systemowy + kontekst + pytanie)
    API->>LLM: POST /api/chat (augmented prompt)
    LLM-->>API: Odpowiedź w języku polskim

    API-->>UI: { "answer": "...", "context_used": [...] }
    UI-->>User: Wyświetla odpowiedź + kontekst
```

### 2. Ścieżka Direct (POST /ask_direct)

```mermaid
sequenceDiagram
    actor User as Użytkownik
    participant UI as Web UI
    participant API as Orchestration API
    participant LLM as Bielik LLM

    User->>UI: Wpisuje pytanie
    UI->>API: POST /ask_direct { "query": "..." }
    API->>LLM: POST /api/chat (surowe pytanie, bez kontekstu)
    LLM-->>API: Odpowiedź w języku polskim
    API-->>UI: { "answer": "..." }
    UI-->>User: Wyświetla odpowiedź (bez kontekstu)
```

### 3. Ścieżka Ingestion (POST /ingest)

```mermaid
sequenceDiagram
    actor Admin as Administrator
    participant UI as Web UI
    participant API as Orchestration API
    participant Emb as Embedding Service
    participant BQ as BigQuery

    Admin->>UI: Przesyła plik CSV
    UI->>API: POST /ingest (multipart CSV)
    loop Dla każdego wiersza CSV
        API->>Emb: POST /api/embed (tekst dokumentu)
        Emb-->>API: wektor 768D
        API->>BQ: INSERT { id, content, embedding }
    end
    API-->>UI: { "status": "success", "inserted_count": N }
    UI-->>Admin: Potwierdzenie załadowania danych
```

---

## Schemat Danych

```mermaid
erDiagram
    HOTEL_RULES {
        STRING id PK "Unikalny identyfikator dokumentu"
        STRING content "Pełna treść reguły (po polsku)"
        FLOAT64 embedding "Wektor 768D (REPEATED)"
    }
```

---

## Topologia Wdrożenia (Google Cloud)

```mermaid
graph LR
    Internet(("🌐 Internet"))

    subgraph GCP["Google Cloud Platform"]
        subgraph CR_PUB["Cloud Run – publiczny"]
            OrchAPI["Orchestration API\norchestration-api\nmaks. 2 instancje"]
        end

        subgraph CR_PRIV["Cloud Run – prywatny (IAM)"]
            EmbSvc["Embedding Service\nembedding-gemma\nmaks. 1 instancja"]
            LLMSvc["Bielik LLM\nbielik\nmaks. 1 instancja · GPU"]
        end

        subgraph BQ_SVC["BigQuery"]
            BQData["rag_dataset.hotel_rules\nVector Search"]
        end

        IAM_SVC["Google Cloud IAM\nIdentity Tokens"]
    end

    Internet -->|"HTTPS"| OrchAPI
    OrchAPI -->|"Bearer token (IAM)"| EmbSvc
    OrchAPI -->|"Bearer token (IAM)"| LLMSvc
    OrchAPI -->|"BigQuery Client Library"| BQData
    OrchAPI -->|"fetch_id_token()"| IAM_SVC
    IAM_SVC -->|"token"| OrchAPI
```
