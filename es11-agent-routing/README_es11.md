# Es.11 – Agent Routing 🚦

## Obiettivo

Costruire un sistema di **routing agentico** in cui un Router Agent analizza l'intent del messaggio Telegram dell'utente e lo smista verso il sotto-agente specializzato corretto, senza che l'utente debba specificare esplicitamente cosa vuole.

---

## Architettura

```
Telegram Trigger
      ↓
Router Agent (Groq, temp=0.0)
  → output: {"route": "meteo"} | {"route": "note"} | {"route": "fermate"} | {"route": "unknown"}
      ↓
Switch Node (JSON.parse($json.output).route)
  ├── meteo   → Agente Meteo   → Send Telegram Message
  ├── note    → Agente Note    → Send Telegram Message
  ├── fermate → Agente Fermate → Send Telegram Message
  └── unknown → Edit Fields    → Send Telegram Message
```

---

## Componenti

### Router Agent
- **Modello:** Groq llama-3.3-70b-versatile
- **Temperature:** 0.0
- **Memory:** Nessuna
- **Tool:** Nessuno
- **Output:** JSON puro `{"route": "<categoria>"}`

**System Message:**
```
Sei un router. Analizza il messaggio dell'utente e rispondi SOLO con un JSON.

Regole:
- Se riguarda meteo, temperatura, pioggia → route: "meteo"
- Se riguarda note, appunti, salvare, ricordare → route: "note"
- Se riguarda fermate, autobus, metro, trasporti Roma → route: "fermate"
- Se non capisci → route: "unknown"

Rispondi SEMPRE e SOLO con questo formato:
{"route": "meteo"}

Nessun testo prima o dopo. Solo JSON.
```

---

### Switch Node
**Expression:**
```
{{ JSON.parse($json.output).route }}
```

| Output | Valore | Destinazione |
|--------|--------|--------------|
| 0 | `meteo` | Agente Meteo |
| 1 | `note` | Agente Note |
| 2 | `fermate` | Agente Fermate |
| 3 | `unknown` | Edit Fields |

---

### Agente Meteo
- **Modello:** Groq llama-3.3-70b-versatile, temp=0.3
- **Memory:** Simple Memory, Context Window=5
- **Tool:** `get_weather` → HTTP GET open-meteo.com
- **Parametri dinamici:** `latitude`, `longitude` via `$fromAI()`

---

### Agente Note
- **Modello:** Groq llama-3.3-70b-versatile, temp=0.3
- **Memory:** Simple Memory, Context Window=5
- **Tool 1:** `save_note` → Google Sheets (appendOrUpdate)
- **Tool 2:** `get_notes` → Google Sheets (read)

---

### Agente Fermate
- **Modello:** Groq llama-3.3-70b-versatile, temp=0.3
- **Memory:** Simple Memory, Context Window=5
- **Tool:** `search_fermate` → Supabase REST (ilike)
- **Link Google Maps** per ogni fermata trovata

---

### Ramo Unknown
- **Nodo:** Edit Fields (Set)
- **Output:** Messaggio statico con suggerimento categorie disponibili
- Nessun AI Agent necessario — risposta predefinita

---

## Espressioni Corrette nei Sub-Agenti

> Nei sottonodi e sub-agenti usare sempre `.first()` invece di `.item.`

**Prompt (User Message):**
```
{{ $('Telegram Trigger').first().json.message.text }}
```

**Simple Memory — Session ID (Key):**
```
{{ $('Telegram Trigger').first().json.message.chat.id }}
```

**Send Telegram — Chat ID:**
```
{{ $('Telegram Trigger').first().json.message.chat.id }}
```

---

## Concetti Chiave

### Routing vs Multi-Agent
| | Routing (Es.11) | Multi-Agent (Es.12) |
|---|---|---|
| **Intent multipli** | Gestisce solo 1 | Gestisce tutti |
| **Agenti attivi** | 1 alla volta | Più in parallelo |
| **Risposta** | Da un solo agente | Consolidata |
| **Complessità** | Semplice | Più complessa |

### .item vs .first()
| Contesto | Sintassi |
|---|---|
| Nodo nel flusso principale | `$('Telegram Trigger').item.json...` |
| Sub-agente / sottonodo | `$('Telegram Trigger').first().json...` |

### Parse
> **Parse** = leggere + interpretare + trasformare dati grezzi in un formato utilizzabile dal computer.
- `JSON.parse()` → stringa JSON → oggetto JavaScript
- Parse CSV → testo → righe e colonne
- Parse PDF → file binario → testo leggibile

---

## Limitazioni

- Un messaggio con **intent multipli** viene gestito su un solo ramo (il primo classificato)
- Esempio: *"Che tempo fa? Salva una nota. Dove si trova la fermata Spagna?"* → il Router sceglie UN solo ramo
- Questa limitazione viene risolta in **Es.12 – Multi-Agent** con un Supervisor che coordina più agenti in parallelo

---

## Stack Tecnologico

| Componente | Tecnologia |
|---|---|
| Trigger | Telegram Bot API |
| LLM | Groq llama-3.3-70b-versatile |
| Meteo | open-meteo.com (API gratuita) |
| Note | Google Sheets |
| Fermate | Supabase (pgvector) da Es.10 |
| Memoria | Simple Memory (n8n nativo) |
| Esposizione locale | ngrok |

---

## Test

| Messaggio | Route attesa |
|---|---|
| `Che tempo fa a Milano?` | `meteo` |
| `Salva una nota: comprare il latte` | `note` |
| `Dove si trova la fermata Colosseo?` | `fermate` |
| `Chi ha vinto la Champions League?` | `unknown` |

---

## Note

- Esercizio precedente: [Es.10 – RAG Avanzato GTFS Roma](../es10-rag-gtfs-roma/)
- Esercizio successivo: [Es.12 – Multi-Agent](../es12-multi-agent/)
