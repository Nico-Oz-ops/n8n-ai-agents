# Es.10 – RAG Avanzato GTFS Roma

Sistema RAG (Retrieval-Augmented Generation) per interrogare in linguaggio naturale le **8.377 fermate del trasporto pubblico di Roma** tramite Telegram, con risposte arricchite da link Google Maps.

---

## Obiettivo

Costruire un AI Agent capace di rispondere a domande come _"Dov'è la fermata del 40 vicino al Colosseo?"_ cercando semanticamente nel database GTFS ufficiale di Roma e restituendo risultati con coordinate e link Maps.

---

## Architettura

```
┌─────────────────────────────────────────────────────────┐
│  WORKFLOW 1 – Ingestion GTFS (eseguito una volta)        │
│                                                          │
│  Manual Trigger                                          │
│       ↓                                                  │
│  Read File from Disk  (stops.txt montato via Docker)     │
│       ↓                                                  │
│  Extract from File    (UTF-8 text)                       │
│       ↓                                                  │
│  Code Node            (parse CSV → 8377 items)           │
│       ↓                                                  │
│  Loop Over Items      (batch size 1)                     │
│       ↓                                                  │
│  HTTP Request OpenAI  (text-embedding-3-small → 1536 dim)│
│       ↓                                                  │
│  HTTP Request Supabase (upsert in gtfs_fermate)          │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  WORKFLOW 2 – AI Agent Telegram (sempre attivo)         │
│                                                         │
│  Telegram Trigger                                       │
│       ↓                                                 │
│  AI Agent ──────────────────────────────────────────┐   │
│  │  Chat Model: GPT-4o-mini (temp 0.3)              │   │
│  │  Memory: Simple Memory (session = chat.id)       │   │
│  └─ Tool: HTTP Request → Supabase RPC               │   │
│       ↓                                             │   │
│  Send Telegram Message ←────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## Tecnologie

| Componente | Tecnologia |
|---|---|
| Orchestratore | n8n self-hosted (Docker) |
| Database vettoriale | Supabase + pgvector |
| Embeddings | OpenAI `text-embedding-3-small` (1536 dim) |
| Chat Model | OpenAI GPT-4o-mini |
| Interfaccia utente | Telegram Bot API |
| Esposizione webhook | ngrok |
| Dati sorgente | GTFS Roma – `stops.txt` (8.377 fermate) |

---

## Prerequisiti

- n8n self-hosted con Docker Compose
- Account Supabase con progetto attivo e estensione `pgvector` abilitata
- API Key OpenAI
- Bot Telegram (creato via @BotFather)
- ngrok attivo per esporre il webhook n8n

---

## Setup Supabase

### 1. Crea la tabella

```sql
create table gtfs_fermate (
  id bigserial primary key,
  stop_id text,
  stop_name text,
  stop_desc text,
  lat float8,
  lon float8,
  content text,
  embedding vector(1536)
);
```

### 2. Crea la funzione di ricerca (RPC)

```sql
create or replace function search_fermate(
  query_embedding vector(1536),
  match_count int default 5
)
returns table (
  id bigint,
  stop_id text,
  stop_name text,
  lat float8,
  lon float8,
  similarity float
)
language plpgsql as $$
begin
  return query
  select
    gtfs_fermate.id,
    gtfs_fermate.stop_id,
    gtfs_fermate.stop_name,
    gtfs_fermate.lat,
    gtfs_fermate.lon,
    1 - (gtfs_fermate.embedding <=> query_embedding) as similarity
  from gtfs_fermate
  order by gtfs_fermate.embedding <=> query_embedding
  limit match_count;
end;
$$;
```

---

## Struttura file

```
es10-rag-gtfs-roma/
├── README.md
├── workflows/
│   ├── workflow1_ingestion_gtfs.json    # Importa in n8n
│   └── workflow2_agent_telegram.json   # Importa in n8n
└── data/
    └── stops.txt                        # GTFS Roma (opzionale, scaricabile da atac.roma.it)
```

---

## Volume Docker per stops.txt

Nel `docker-compose.yml`, aggiungi il mount del file GTFS al container n8n:

```yaml
volumes:
  - /home/its/VsCode_Project/es10-rag-gtfs-roma/data:/home/node/.n8n-files/gtfs
```

Il nodo `Read/Write Files from Disk` leggerà da:
```
/home/node/.n8n-files/gtfs/stops.txt
```

---

## Configurazione AI Agent (Workflow 2)

### System Message

```
Sei un assistente esperto di trasporto pubblico di Roma. Aiuti gli utenti
a trovare fermate dell'autobus e della metro usando il database GTFS di Roma.
Per ogni fermata trovata, mostra sempre un link Google Maps usando le coordinate
lat e lon nel formato: https://www.google.com/maps?q=LAT,LON.
Rispondi sempre in italiano in modo chiaro e conciso.
```

### HTTP Request Tool – parametri chiave

| Parametro | Valore |
|---|---|
| Method | GET |
| URL | `https://<project>.supabase.co/rest/v1/gtfs_fermate` |
| Query: `select` | `stop_id,stop_name,lat,lon` |
| Query: `stop_name` | `ilike.*{{ $fromAI('fermata') }}*` |
| Query: `limit` | `5` |
| Header: `Prefer` | `return=representation` |

---

## Esempi di utilizzo

```
Utente: dove si trova la fermata termini
Agent:  Ecco le fermate vicino a Termini:
        • 70095 – TERMINI (41.8990, 12.5011) → Maps
        • 70096 – TERMINI/TIBURTINA ...

Utente: fermate della metro A vicino al vaticano
Agent:  Ho trovato queste fermate:
        • OTTAVIANO – S.PIETRO/MUSEI VATICANI (41.9037, 12.4568) → Maps
```

---

## Note tecniche

### Sintassi ilike Supabase REST
Per la ricerca case-insensitive con wildcard in Supabase REST API, usare **asterischi** (non `%`):
```
stop_name=ilike.*termine*
```

### $fromAI() in n8n
Sintassi n8n per permettere all'AI Agent di popolare dinamicamente i parametri del Tool:
```
{{ $fromAI('fermata', 'Nome della fermata da cercare') }}
```
Il parametro `fermata` viene inferito dall'Agent in base al messaggio dell'utente.

### Ingestion: perché batch size 1
Il Loop Over Items con batch 1 garantisce che ogni embedding sia associato alla fermata corretta. Batch più grandi richiedono gestione manuale dell'indice.

### Costo stimato ingestion
8.377 chiamate a `text-embedding-3-small` ≈ **$0.003** (ottobre 2024).

---

## Limitazioni attuali

Il database indicizzato contiene solo `stops.txt` (8.377 fermate).
Il sistema può trovare fermate per nome e posizione, ma **non conosce**:

- Le linee che passano per ogni fermata (`routes.txt`)
- Gli orari di passaggio (`stop_times.txt` — ~5,5 milioni di righe)

Per domande tipo "quale autobus prendo?" o "a che ora passa il prossimo 40?", l'agent suggerisce Google Maps o Moovit come alternative.

> Nota: fare embedding su `stop_times.txt` non è la soluzione giusta — si tratta di dati strutturati, non semantici. La risposta corretta è una query SQL con filtri su orario e linea, non una similarity search.

## Roadmap

Queste limitazioni vengono affrontate in **Es.14 – Smart Commute Roma** (esercizio finale), dove verranno integrati:

- `routes.txt` → per sapere quale linea passa per ogni fermata
- `stop_times.txt` → per rispondere a "a che ora passa il prossimo autobus?" e "c'è una corsa diretta tra le 7:30 e le 8:30?"

L'approccio sarà **query SQL diretta su Supabase** (senza embedding), molto più efficiente per dati tabellari di questa dimensione.

---

## Fase di appartenenza

Fase 3 – Knowledge & Retrieval  
Percorso: [n8n AI Agents – Percorso Completo](../README.md)
