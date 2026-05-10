# Es.13 – Personal Assistant

**Fase 4 – Architettura Agentica**  
Pattern: Supervisor + 6 Worker Agents  
Interfaccia: n8n Chat + Telegram

---

## Descrizione

Personal Assistant completo basato sul pattern Supervisor + Worker Agents (introdotto in Es.12).  
Il Supervisor riceve i messaggi dell'utente e delega a 6 worker specializzati, ciascuno con competenze specifiche.

Novità rispetto a Es.12:
- Integrazione Google completa (Gmail, Calendar, Contacts, Drive)
- Ricerca web via SerpAPI
- Worker RAG con knowledge base sui documenti del corso IA (Supabase pgvector + OpenAI embeddings)

---

## Architettura

```
Chat n8n / Telegram
        ↓
   es13-supervisor (gpt-4o-mini, Simple Memory)
        ↓
   Call n8n Workflow
   ├── email_worker      → Gmail (leggi, invia, rispondi)
   ├── calendar_worker   → Google Calendar (leggi, crea, elimina)
   ├── contacts_worker   → Google Contacts (cerca, crea, aggiorna)
   ├── writer_worker     → Solo LLM (scrittura creativa)
   ├── meteo_worker      → open-meteo.com (HTTP Request)
   └── knowledge_worker  → RAG (Supabase pgvector + OpenAI embeddings)
```

---

## Workflow

| File | Descrizione |
|------|-------------|
| `es13-supervisor` | Supervisor principale |
| `es13-worker-email` | Worker Gmail |
| `es13-worker-calendar` | Worker Google Calendar |
| `es13-worker-contacts` | Worker Google Contacts |
| `es13-worker-writer` | Worker scrittura |
| `es13-worker-meteo` | Worker meteo |
| `es13-worker-knowledge` | Worker RAG knowledge base |
| `es13-ingestion-knowledge` | Workflow ingestion PDF (one-shot) |

---

## Worker Knowledge — RAG

Il componente più articolato di Es.13. Implementa un sistema RAG completo:

### Knowledge Base
- **Fonte**: Google Drive, cartella IA_8
- **Documenti**: 6 PDF del corso Professional Data Analyst
- **Chunk indicizzati**: 232
- **Tabella Supabase**: `ia_documents`

### Stack tecnico
- **Embeddings**: OpenAI `text-embedding-3-small` (1536 dimensioni)
- **Vector DB**: Supabase pgvector
- **Chunk size**: 1000 caratteri, overlap 200
- **Retrieval**: top 4 chunk per similarità coseno

### Flusso RAG

**Ingestion (one-shot):**
```
PDF → Google Drive Download → Default Data Loader (chunk) → OpenAI Embeddings → Supabase INSERT
```

**Query (real-time):**
```
Domanda utente → OpenAI Embeddings → vettore domanda
→ Supabase match_documents → top 4 chunk simili
→ gpt-4o-mini → risposta in italiano
```

### Setup Supabase

```sql
-- Tabella
create extension if not exists vector;
create table ia_documents (
  id bigserial primary key,
  content text,
  metadata jsonb,
  embedding vector(1536)
);
create index on ia_documents
using ivfflat (embedding vector_cosine_ops)
with (lists = 100);

-- Funzione di ricerca
create or replace function match_documents (
  query_embedding vector(1536),
  match_count int default 4,
  filter jsonb default '{}'
)
returns table (id bigint, content text, metadata jsonb, similarity float)
language plpgsql as $$
begin
  return query
  select ia_documents.id, ia_documents.content, ia_documents.metadata,
         1 - (ia_documents.embedding <=> query_embedding) as similarity
  from ia_documents
  where ia_documents.metadata @> filter
  order by ia_documents.embedding <=> query_embedding
  limit match_count;
end; $$;
```

### Aggiornamento Knowledge Base

- **Aggiungere nuovi PDF**: caricare su Drive e rilasciare l'ingestion (INSERT, non tocca dati esistenti)
- **Aggiornare un PDF**: prima `DELETE FROM ia_documents WHERE metadata->>'source' = 'nome.pdf'`, poi rilasciare l'ingestion

---

## Configurazione Worker

| Worker | Modello | Temp | MaxIter | Memory |
|--------|---------|------|---------|--------|
| supervisor | gpt-4o-mini | 0.3 | 10 | Simple Memory |
| email | gpt-4o-mini | 0.3 | 5 | No |
| calendar | gpt-4o-mini | 0.3 | 5 | No |
| contacts | gpt-4o-mini | 0.3 | 5 | No |
| writer | gpt-4o-mini | 0.5 | 5 | No |
| meteo | gpt-4o-mini | 0.3 | 5 | No |
| knowledge | gpt-4o-mini | 0.3 | 5 | No |

---

## Connessioni OAuth

- Gmail ✅
- Google Calendar ✅
- Google Contacts ✅
- Google Drive ✅
- OpenAI ✅
- Supabase ✅
- SerpAPI ✅

---

## Regole Architetturali

- Solo il Supervisor ha memoria (Simple Memory, sessionId dinamico)
- I worker usano `When Executed by Another Workflow` come trigger
- Il Supervisor usa `Call n8n Workflow` come tool
- NON usare `Respond to Webhook` nei worker
- I worker devono essere Active (Published)
- Testare SEMPRE via interfaccia reale, mai con Execute manuale
- Nei sub-agenti usare sempre `.first()` invece di `.item`
- Il campo `Fields` in Google Contacts Get Many va selezionato manualmente (NON defined by model)
- I campi `After`/`Before` in Google Calendar Get Many → defined by model
- Il campo `To` in Gmail Send → defined by model

---

## Infrastruttura

- n8n su Docker (VM Ubuntu 22.04, 15GB RAM)
- ngrok per webhook Telegram
- Supabase (pgvector) per knowledge base
- OpenAI API (gpt-4o-mini + text-embedding-3-small)

---

## Concetti Chiave

- **RAG** (Retrieval Augmented Generation): arricchisce l'AI con documenti esterni
- **Embeddings**: rappresentazione vettoriale del significato semantico del testo
- **Similarità coseno**: misura la vicinanza semantica tra due vettori
- **pgvector**: estensione PostgreSQL per ricerca vettoriale
- **Chunking**: divisione del testo in pezzi gestibili per l'indicizzazione
