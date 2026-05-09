# Es.12 – Multi-Agent: Supervisor + Worker

## Obiettivo

Gestire **intent multipli in un singolo messaggio Telegram** usando il pattern **Supervisor + Worker Agent**.

Il Supervisor analizza il messaggio, identifica tutti gli intent e coordina i worker specializzati, consolidando le risposte in un unico messaggio.

---

## Architettura

```
Telegram → es12-supervisor (AI Agent)
               ├── Call n8n Workflow → es12-worker-meteo
               ├── Call n8n Workflow → es12-worker-note
               └── Call n8n Workflow → es12-worker-fermate
                        ↓
               Risposta consolidata → Telegram
```

---

## File del progetto

| File | Trigger | Descrizione |
|------|---------|-------------|
| `es12-supervisor` | Telegram Trigger | Riceve il messaggio, coordina i worker, consolida la risposta |
| `es12-worker-meteo` | Execute Workflow Trigger | Agente specializzato meteo (open-meteo API) |
| `es12-worker-note` | Execute Workflow Trigger | Agente specializzato note (Google Sheets) |
| `es12-worker-fermate` | Execute Workflow Trigger | Agente specializzato fermate Roma (Supabase) |

---

## es12-supervisor

**Struttura:** `Telegram Trigger → AI Agent Supervisor → Send Telegram Message`

**Configurazione AI Agent:**
- Chat Model: OpenAI `gpt-4o-mini`
- Temperature: `0.3`
- Max Iterations: `10`
- Memory: Simple Memory (`sessionId = chat.id`)
- Prompt: `{{ $('Telegram Trigger').first().json.message.text }}`

**System Message:**
```
Sei un assistente personale che risponde SEMPRE in italiano.

REGOLA FONDAMENTALE: non rispondere mai direttamente.
Usa SEMPRE i tool disponibili per soddisfare le richieste.

- Per richieste meteo: usa SEMPRE get_weather
- Per note: usa SEMPRE manage_notes
- Per fermate bus/metro: usa SEMPRE search_fermate

Esegui TUTTI i tool necessari, poi consolida le risposte
in un unico messaggio chiaro.
```

**Tool (Call n8n Workflow):**

| Tool | Workflow | Campo | fromAI |
|------|----------|-------|--------|
| `get_weather` | es12-worker-meteo | `query` | `$fromAI('query', 'la richiesta meteo')` |
| `manage_notes` | es12-worker-note | `query` | `$fromAI('query', 'il testo della nota')` |
| `fermate` | es12-worker-fermate | `query` | `$fromAI('query', 'nome fermata da cercare')` |

---

## es12-worker-meteo

**Struttura:** `When Executed by Another Workflow → AI Agent Meteo`

**Configurazione:**
- Trigger Input: campo `query` (String)
- Prompt: `{{ $json.query }}`
- Temperature: `0.0` | Max Iterations: `5` | No Memory

**System Message:**
```
Sei un esperto meteo. Usa il tool get_weather per ottenere
le condizioni meteo della città richiesta.

Se l'utente chiede il meteo attuale, usa i dati 'current'.
Se l'utente chiede le previsioni future, usa i dati 'daily'
e presentali giorno per giorno in italiano con temperatura
massima, minima, condizioni e precipitazioni.
```

**Tool HTTP Request (get_weather):**

| Parametro | Valore |
|-----------|--------|
| Method | GET |
| URL | `https://api.open-meteo.com/v1/forecast` |
| `latitude` | `$fromAI('latitude', 'latitudine della città')` |
| `longitude` | `$fromAI('longitude', 'longitudine della città')` |
| `current` | `temperature_2m,weathercode,windspeed_10m` |
| `daily` | `temperature_2m_max,temperature_2m_min,weathercode,precipitation_sum` |
| `forecast_days` | `7` |
| `timezone` | `auto` |

> Il parametro `daily` e `forecast_days` permettono all'agente di rispondere anche a previsioni future. L'AI decide autonomamente quali dati usare in base alla domanda:
> - "Che tempo fa a Roma?" → usa `current`
> - "Previsioni per i prossimi 3 giorni a Milano?" → usa `daily`

> ⚠️ **NON aggiungere Respond to Webhook.** L'AI Agent è l'ultimo nodo. n8n restituisce automaticamente il suo output al Supervisor.

---

## es12-worker-note

**Struttura:** `When Executed by Another Workflow → AI Agent Note`

**Configurazione:**
- Trigger Input: campo `query` (String)
- Prompt: `{{ $json.query }}`
- Temperature: `0.0` | Max Iterations: `5` | No Memory

**System Message:**
```
Sei un assistente per la gestione delle note.
Usa save_note per salvare nuove note su Google Sheets.
Usa get_notes per recuperare le note già salvate.
Rispondi in italiano confermando l'operazione eseguita
o elencando le note trovate.
```

**Tool save_note (Google Sheets – Append Row):**
- `title`: `$fromAI('title', 'titolo della nota')`
- `content`: `$fromAI('content', 'contenuto della nota')`
- `savedAt`: `{{ new Date().toISOString() }}`

**Tool get_notes (Google Sheets – Get Many Rows):** nessun parametro aggiuntivo.

> ⚠️ **NON aggiungere Respond to Webhook.**

---

## es12-worker-fermate

**Struttura:** `When Executed by Another Workflow → AI Agent Fermate`

**Configurazione:**
- Trigger Input: campo `query` (String)
- Prompt: `{{ $json.query }}`
- Temperature: `0.0` | Max Iterations: `5` | No Memory

> Il system message e il tool Supabase sono gli stessi di Es.10 e Es.11.

> ⚠️ **NON aggiungere Respond to Webhook.**

---

## Differenza Es.11 vs Es.12

| | Es.11 – Router | Es.12 – Multi-Agent |
|---|---|---|
| Architettura | Switch Node | Supervisor Agent |
| Intent gestiti | 1 solo per messaggio | Tutti quelli presenti |
| Agenti attivi | 1 alla volta | Tutti i necessari |
| Coordinazione | Rigida (Switch) | Autonoma (AI decide) |

---

## Concetti chiave appresi

- **Supervisor pattern**: un agente che orchestra altri agenti
- **Worker Agent**: agente specializzato, chiamato come tool
- **Call n8n Workflow**: tool per richiamare workflow separati
- **Execute Workflow Trigger**: come i worker ricevono le chiamate
- **Intent multipli**: il Supervisor gestisce tutto in un messaggio
- **Ragionamento contestuale**: l'agente decide autonomamente se chiamare un tool
- **Respond to Webhook ≠ Execute Workflow**: pattern diversi, non intercambiabili
- **Worker senza memory**: ogni chiamata è stateless, solo il Supervisor ha memoria

---

## Errori comuni e soluzioni

| Errore | Causa | Soluzione |
|--------|-------|-----------|
| `Workflow is not active` | Worker non attivato | Attivare toggle Active su ogni worker |
| `No Webhook node found` | Respond to Webhook nel worker | Eliminarlo, lasciare AI Agent come ultimo nodo |
| `messages.1.content empty` | Test manuale senza Telegram | Testare solo via Telegram, mai con Execute manuale |
| Rate limit Groq | Troppi token con multi-agent | Usare gpt-4o-mini per i test |
| Più Execute Workflow Trigger | Limite n8n su stesso canvas | Creare workflow separati per ogni worker |

---

## Stack tecnico

- **Trigger**: Telegram Bot API
- **Supervisor**: OpenAI gpt-4o-mini, temp 0.3, Simple Memory
- **Worker Meteo**: open-meteo.com API (meteo attuale + previsioni 7 giorni)
- **Worker Note**: Google Sheets (save + get)
- **Worker Fermate**: Supabase REST API (pgvector, dati GTFS Roma)
- **Tool di collegamento**: Call n8n Workflow
