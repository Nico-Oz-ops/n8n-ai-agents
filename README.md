# n8n AI Agents – Percorso Completo

Percorso pratico di apprendimento per costruire **AI Agent con n8n**, da zero fino a un sistema multi-agent completo integrato con Telegram, Google Workspace e dati real-world.

14 esercizi · 5 fasi · self-hosted · no-code first

---

## Panoramica del percorso

```
Fase 1 – Fondamenta          (Es.01–05)  ✅
Fase 2 – Controllo           (Es.06–08)  ✅
Fase 3 – Knowledge & RAG     (Es.09–10)  ✅
Fase 4 – Architettura agent  (Es.11–12)  ✅
Fase 5 – Integrazione reale  (Es.13–14)  🔄
```

---

## Esercizi

| # | Nome | Stato | Descrizione |
|---|------|-------|-------------|
| Es.01 | Tool Calling Base | ✅ | Code Tool JS, variabile `query`, output stringa |
| Es.02 | System Message | ✅ | Italiano, max 2 frasi, tono professionale |
| Es.03 | Conversational Memory | ✅ | Simple Memory, sessionId, Context Window 5 |
| Es.04 | Tool con Parametri | ✅ | HTTP Request → open-meteo.com, AI inferisce lat/lon |
| Es.05 | Memoria Persistente | ✅ | Google Sheets, tool `save_note`/`get_notes` |
| Es.06 | Condizioni e Branching | ✅ | Switch Node, output JSON con `category` e `summary` |
| Es.07 | Prompt Engineering | ✅ | Few-shot, structured output, sentiment analysis |
| Es.08 | Multi-tool Coordination | ✅ | Telegram Trigger, meteo + note su Sheets |
| Es.09 | RAG Semplice | ✅ | Naive RAG e Keyword RAG su Google Sheets |
| Es.10 | RAG Avanzato GTFS Roma | ✅ | pgvector, OpenAI embeddings, 8.377 fermate ATAC |
| Es.11 | Agent Routing | ✅ | Router agent → worker specializzati |
| Es.12 | Multi-Agent | ✅ | Supervisor + worker agents |
| Es.13 | API con Autenticazione | ⏳ | OAuth2, token refresh, API sicure |
| Es.14 | Smart Commute Roma | ⏳ | Sistema finale end-to-end |

---

## Tecnologie utilizzate

| Categoria | Tecnologia |
|---|---|
| Orchestratore | n8n self-hosted (Docker) |
| LLM principale | Groq `llama-3.3-70b-versatile` |
| LLM secondario | OpenAI GPT-4o-mini |
| Embeddings | OpenAI `text-embedding-3-small` |
| Database vettoriale | Supabase + pgvector |
| Memoria conversazionale | n8n Simple Memory |
| Storage note | Google Sheets |
| Interfaccia utente | Telegram Bot API |
| Esposizione webhook | ngrok |
| Dati trasporti | GTFS Roma (ATAC) |

---

## Ambiente di sviluppo

```
OS:         Ubuntu 22.04
RAM:        15 GB
Runtime:    Docker Compose
n8n path:   /home/its/sw_development/
Shared vol: /home/its/VsCode_Project → /shared (nei container)
LLM locale: Groq API (nessuna GPU necessaria)
Webhook:    ngrok → porta n8n (default 5678)
```

### docker-compose.yml (estratto rilevante)

```yaml
services:
  n8n:
    image: n8nio/n8n
    ports:
      - "5678:5678"
    volumes:
      - /home/its/VsCode_Project:/shared
      - n8n_data:/home/node/.n8n
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - WEBHOOK_URL=https://<tuo-ngrok-id>.ngrok.io
```

---

## Come usare questa repo

### 1. Clona la repo

```bash
git clone https://github.com/<tuo-username>/n8n-ai-agents.git
cd n8n-ai-agents
```

### 2. Avvia n8n

```bash
cd /home/its/sw_development
docker compose up -d
```

### 3. Importa un workflow

1. Apri n8n su `http://localhost:5678`
2. Vai su **Workflows → Import from file**
3. Seleziona il `.json` nella cartella dell'esercizio

### 4. Configura le credenziali

Ogni esercizio richiede credenziali diverse. Controlla il README nella cartella dell'esercizio per i dettagli. Le credenziali comuni da configurare:

- Groq API Key
- OpenAI API Key
- Google Sheets OAuth2
- Telegram Bot Token
- Supabase API Key + URL

---

## Struttura della repo

```
n8n-ai-agents/
├── README.md                          # Questo file
├── es01-tool-calling/
│   ├── README.md
│   └── workflows/
├── es02-system-message/
├── es03-memory/
├── es04-http-tool/
├── es05-persistent-memory/
├── es06-branching/
├── es07-prompt-engineering/
├── es08-multi-tool/
├── es09-rag-base/
├── es10-rag-gtfs-roma/               # RAG avanzato con pgvector
│   ├── README.md
│   ├── workflows/
│   └── data/
├── es11-agent-routing/               # 🔄 in corso
├── es12-multi-agent/                 # ⏳
├── es13-api-auth/                    # ⏳
└── es14-smart-commute-roma/          # ⏳ – esercizio finale
```

---

## Principi guida

- **Tool nativi prima di tutto** — usare nodi n8n nativi (Google Sheets, Gmail, Drive) invece del Code Tool quando possibile
- **Semplicità prima della perfezione** — arrivare alla soluzione più semplice e robusta
- **Step-by-step** — ogni esercizio consolida i concetti del precedente
- **Real data** — dati reali (GTFS, meteo, Gmail) fin dall'inizio

---

## Note importanti

### `fs` non disponibile nel Code Tool
n8n Code Tool gira in un ambiente sandbox senza accesso al filesystem. Usare il nodo `Read/Write Files from Disk` per operazioni su file.

### `$getWorkflowStaticData` non persiste tra esecuzioni
Per la persistenza dei dati usare Google Sheets, Supabase o altri storage esterni.

### ngrok e webhook Telegram
Telegram richiede un URL HTTPS pubblico per i webhook. ngrok espone n8n in locale. Ricordare di aggiornare `WEBHOOK_URL` nel docker-compose quando l'URL ngrok cambia (a meno di usare un dominio fisso con ngrok paid).

---

## Esercizio finale – Es.14 Smart Commute Roma

Il progetto culminante del percorso: un sistema completo che integra tutte le tecniche apprese per creare un assistente personale per i pendolari romani. Dettagli nella cartella dedicata una volta completati gli esercizi precedenti.

---

## Licenza

MIT – uso libero per apprendimento e progetti personali.
