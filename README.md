# "Autonomous AI Agent & Micro-SaaS Starter Kit" — Un template production-ready open-core (FastAPI + Next.js 14 + Supabase + Stripe/Gumroad webhook integration) che permette agli sviluppatori di rilasciare agenti AI autonomi monetizzabili in meno di un'ora. La versione base/demo viene rilasciata open source su GitHub con documentazione curata, mentre la versione Premium (con orchestrazione multi-agente avanzata, auth, dashboard di fatturazione e workflow n8n inclusi) viene venduta su Gumroad a 29€.

Ecco il pacchetto completo, progettato secondo l'approccio **Open-Core**: architettura solida, codice pronto per la produzione, funnel di conversione integrato e blueprint di lancio.

---

# 📦 STRUTTURA DEL REPOSITORY (File Tree)

```text
agent-saas-starterkit/
├── .github/workflows/ci.yml
├── backend/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── agent/
│   │   │   ├── __init__.py
│   │   │   ├── runner.py
│   │   │   └── tools.py
│   │   ├── db/
│   │   │   ├── __init__.py
│   │   │   └── supabase_client.py
│   │   └── api/
│   │       ├── v1/
│   │       │   ├── agent.py
│   │       │   └── webhooks.py
│   ├── requirements.txt
│   └── Dockerfile
├── frontend/
│   ├── app/
│   │   ├── api/agent/route.ts
│   │   ├── dashboard/page.tsx
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── components/
│   │   └── AgentTerminal.tsx
│   ├── package.json
│   └── tailwind.config.ts
├── supabase/
│   └── migrations/20240101_init.sql
├── .env.example
├── README.md
└── LICENSE
```

---

# 1. README.md (Open Source + Upsell Copy)

```markdown
# ⚡ AgentSaaS: The Autonomous AI Agent Micro-SaaS Starter Kit

> Launch monetizable, autonomous AI agents in under 60 minutes. Built for solopreneurs, indie hackers, and AI engineers.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.110+-009688.svg?logo=fastapi)](https://fastapi.tiangolo.com)
[![Next.js 14](https://img.shields.io/badge/Next.js-14_App_Router-black.svg?logo=next.js)](https://nextjs.org)
[![Supabase](https://img.shields.io/badge/Database-Supabase-3ECF8E.svg?logo=supabase)](https://supabase.com)

---

## 🎯 What is AgentSaaS?

Building AI agent logic is fast, but turning it into a **cash-flowing business** takes weeks of boilerplate: authentication, usage limits, webhook handlers, customer dashboards, and deployment.

**AgentSaaS** solves this. It's an **Open-Core** blueprint combining a resilient **FastAPI** autonomous agent loop, **Supabase** (Auth + DB + RLS), **Next.js 14**, and instant monetization via **Gumroad** and **Stripe**.

---

## ⚡ Free vs. Premium Edition

| Feature | Open Source (This Repo) | 🚀 Premium PRO Edition (€29) |
| :--- | :---: | :---: |
| **Agent Engine** | Single ReAct Agent (OpenAI/Groq) | Multi-Agent Swarm (Planner + Worker + Critic) |
| **Monetization** | Webhook Gumroad & Stripe Base | Auto-sync Tier Subscription + Credit Top-ups |
| **Database** | Supabase Base Schema | Advanced Schema + Vector Store (pgvector) |
| **Frontend** | Minimal Playground UI | Full Dashboard UI, Dark Mode, Usage Graphs |
| **Automation** | Manual execution | Pre-built n8n workflows for auto-marketing |
| **Support** | Community via GitHub Issues | Private Discord + Direct Updates |

👉 **[Unlock the Premium Edition on Gumroad for €29](https://py-forge.com)** *(Includes full production license & priority access)*

---

## 🛠️ Quickstart (Local Dev)

### 1. Clona il repository
```bash
git clone https://github.com/your-username/agent-saas-starterkit.git
cd agent-saas-starterkit
```

### 2. Configura le variabili d'ambiente
```bash
cp .env.example backend/.env
cp .env.example frontend/.env.local
```

### 3. Setup Backend (FastAPI)
```bash
cd backend
python -m venv venv
source venv/bin/activate # Windows: venv\Scripts\activate
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
```

### 4. Setup Frontend (Next.js)
```bash
cd ../frontend
npm install
npm run dev
```

Apri `http://localhost:3000` nel browser.
```

---

# 2. SCHEMA DEL DATABASE (Supabase SQL)

Salva in `supabase/migrations/20240101_init.sql`:

```sql
-- Abilita l'estensione UUID
create extension if not exists "uuid-ossp";

-- Tabella Profili Utente
create table public.profiles (
    id uuid references auth.users on delete cascade primary key,
    email text unique not null,
    is_premium boolean default false,
    credits integer default 10,
    created_at timestamp with time zone default timezone('utc'::text, now()) not null
);

-- Tabella Sessioni/Task dell'Agente
create table public.agent_tasks (
    id uuid default uuid_generate_v4() primary key,
    user_id uuid references public.profiles(id) on delete cascade not null,
    prompt text not null,
    status text check (status in ('pending', 'running', 'completed', 'failed')) default 'pending',
    output jsonb,
    tokens_used integer default 0,
    created_at timestamp with time zone default timezone('utc'::text, now()) not null
);

-- Tabella Log Pagamenti Webhook (Stripe/Gumroad)
create table public.payment_events (
    id uuid default uuid_generate_v4() primary key,
    event_id text unique not null,
    provider text not null, -- 'stripe' o 'gumroad'
    payload jsonb not null,
    processed boolean default false,
    created_at timestamp with time zone default timezone('utc'::text, now()) not null
);

-- Attiva Row Level Security (RLS)
alter table public.profiles enable row level security;
alter table public.agent_tasks enable row level security;

-- Policies RLS
create policy "Gli utenti possono leggere il proprio profilo"
    on public.profiles for select using (auth.uid() = id);

create policy "Gli utenti possono leggere i propri task"
    on public.agent_tasks for select using (auth.uid() = user_id);

create policy "Gli utenti possono creare task se hanno crediti"
    on public.agent_tasks for insert with check (
        auth.uid() = user_id and 
        exists (
            select 1 from public.profiles 
            where id = auth.uid() and credits > 0
        )
    );
```

---

# 3. BACKEND CORE (FastAPI + Agent Engine)

### `backend/requirements.txt`
```text
fastapi>=0.110.0
uvicorn>=0.28.0
pydantic>=2.6.0
pydantic-settings>=2.2.1
openai>=1.14.0
supabase>=2.3.0
stripe>=8.5.0
python-dotenv>=1.0.1
requests>=2.31.0
```

### `backend/app/config.py`
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    OPENAI_API_KEY: str
    SUPABASE_URL: str
    SUPABASE_SERVICE_ROLE_KEY: str
    STRIPE_WEBHOOK_SECRET: str
    GUMROAD_PRODUCT_ID: str = ""
    PORT: int = 8000

    class Config:
        env_file = ".env"
        extra = "ignore"

settings = Settings()
```

### `backend/app/db/supabase_client.py`
```python
from supabase import create_client, Client
from app.config import settings

def get_supabase_admin() -> Client:
    """Ritorna il client con privilegi amministrativi (bypassa RLS per webhooks/cron)."""
    return create_client(settings.SUPABASE_URL, settings.SUPABASE_SERVICE_ROLE_KEY)
```

### `backend/app/agent/runner.py`
```python
import json
from openai import OpenAI
from app.config import settings

client = OpenAI(api_key=settings.OPENAI_API_KEY)

# Tool di esempio per esecuzione autonoma
def execute_python_code(code: str) -> str:
    """Esegue codice Python semplice in modo controllato (simulazione)."""
    # ATTENZIONE: In produzione usa un runtime isolato come E2B o Docker sandbox
    return f"Risultato elaborazione: codice '{code[:30]}...' eseguito con successo."

TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "execute_python_code",
            "description": "Esegue codice Python per calcoli o analisi dati.",
            "parameters": {
                "type": "object",
                "properties": {
                    "code": {"type": "string", "description": "Il codice Python da eseguire."}
                },
                "required": ["code"]
            }
        }
    }
]

async def run_autonomous_loop(objective: str) -> dict:
    """Ciclo ReAct di base per l'agente autonomo."""
    messages = [
        {"role": "system", "content": "Sei un agente AI economico ed esecutivo autonomo. "
                                      "Raggiungi l'obiettivo dell'utente usando i tool a disposizione. "
                                      "Sii estremamente conciso e orientato ai risultati."},
        {"role": "user", "content": objective}
    ]

    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=messages,
        tools=TOOLS,
        tool_choice="auto"
    )

    response_message = response.choices[0].message
    total_tokens = response.usage.total_tokens

    # Verifica se l'agente ha invocato un tool
    if response_message.tool_calls:
        tool_call = response_message.tool_calls[0]
        function_name = tool_call.function.name
        arguments = json.loads(tool_call.function.arguments)

        if function_name == "execute_python_code":
            tool_output = execute_python_code(arguments.get("code", ""))
            
            # Secondo turno per completare la risposta post-tool
            messages.append(response_message)
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": tool_output
            })
            
            final_response = client.chat.completions.create(
                model="gpt-4o-mini",
                messages=messages
            )
            return {
                "status": "completed",
                "output": final_response.choices[0].message.content,
                "tokens": total_tokens + final_response.usage.total_tokens
            }

    return {
        "status": "completed",
        "output": response_message.content,
        "tokens": total_tokens
    }
```

### `backend/app/api/v1/webhooks.py`
```python
import stripe
from fastapi import APIRouter, Request, Header, HTTPException, Form
from app.config import settings
from app.db.supabase_client import get_supabase_admin

router = APIRouter()
supabase = get_supabase_admin()

@router.post("/stripe")
async def stripe_webhook(request: Request, stripe_signature: str = Header(None)):
    """Gestione eventi Stripe: aggiorna lo status utente su Supabase."""
    payload = await request.body()
    try:
        event = stripe.Webhook.construct_event(
            payload, stripe_signature, settings.STRIPE_WEBHOOK_SECRET
        )
    except Exception as e:
        raise HTTPException(status_code=400, detail=f"Errore validazione Stripe: {str(e)}")

    if event["type"] == "checkout.session.completed":
        session = event["data"]["object"]
        customer_email = session.get("customer_details", {}).get("email")

        if customer_email:
            # Upgrade dell'account e aggiunta crediti
            supabase.table("profiles").update({
                "is_premium": True,
                "credits": 500
            }).eq("email", customer_email).execute()

    return {"status": "success"}

@router.post("/gumroad")
async def gumroad_webhook(
    email: str = Form(...),
    product_id: str = Form(...),
    sale_id: str = Form(...)
):
    """Gestione webhook Gumroad (inviato in application/x-www-form-urlencoded)."""
    # Salva l'evento per idempotenza
    try:
        supabase.table("payment_events").insert({
            "event_id": sale_id,
            "provider": "gumroad",
            "payload": {"email": email, "product_id": product_id}
        }).execute()
    except Exception:
        return {"status": "ignored", "reason": "already_processed"}

    # Aggiorna utente
    supabase.table("profiles").update({
        "is_premium": True,
        "credits": 250
    }).eq("email", email).execute()

    return {"status": "success", "message": "Gumroad sale parsed"}
```

### `backend/app/main.py`
```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from app.agent.runner import run_autonomous_loop
from app.db.supabase_client import get_supabase_admin
from app.api.v1.webhooks import router as webhook_router

app = FastAPI(title="AgentSaaS Core Engine")
app.include_router(webhook_router, prefix="/api/v1/webhooks", tags=["Webhooks"])
supabase = get_supabase_admin()

class TaskRequest(BaseModel):
    user_id: str
    prompt: str

@app.post("/api/v1/agent/run")
async def run_agent_task(req: TaskRequest):
    # Verifica crediti
    res = supabase.table("profiles").select("credits").eq("id", req.user_id).execute()
    if not res.data or res.data[0]["credits"] <= 0:
        raise HTTPException(status_code=402, detail="Crediti insufficienti. Effettua l'upgrade.")

    # Esegui l'agente
    result = await run_autonomous_loop(req.prompt)

    # Scala crediti e logga task
    supabase.table("profiles").update({
        "credits": res.data[0]["credits"] - 1
    }).eq("id", req.user_id).execute()

    supabase.table("agent_tasks").insert({
        "user_id": req.user_id,
        "prompt": req.prompt,
        "status": result["status"],
        "output": {"result": result["output"]},
        "tokens_used": result["tokens"]
    }).execute()

    return result
```

---

# 4. FRONTEND INTERFACE (Next.js 14 App Router)

### `frontend/components/AgentTerminal.tsx`
```tsx
'use client';

import React, { useState } from 'react';
import { Terminal, Send, Zap } from 'lucide-react';

export default function AgentTerminal({ userId }: { userId: string }) {
  const [prompt, setPrompt] = useState('');
  const [loading, setLoading] = useState(false);
  const [logs, setLogs] = useState<string[]>([]);

  const handleRun = async () => {
    if (!prompt) return;
    setLoading(true);
    setLogs((prev) => [...prev, `> Obiettivo inviato: "${prompt}"`, '> Agente in esecuzione...']);

    try {
      const res = await fetch('http://localhost:8000/api/v1/agent/run', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ user_id: userId, prompt }),
      });

      if (!res.ok) {
        if (res.status === 402) {
          setLogs((prev) => [...prev, '[ERRORE]: Crediti esauriti. Fai l\'upgrade su Gumroad!']);
          return;
        }
        throw new Error('Errore durante l\'esecuzione');
      }

      const data = await res.json();
      setLogs((prev) => [...prev, `[OUTPUT]: ${data.output}`, `[INFO]: Token consumati: ${data.tokens}`]);
    } catch (err: any) {
      setLogs((prev) => [...prev, `[FATAL]: ${err.message}`]);
    } finally {
      setLoading(false);
      setPrompt('');
    }
  };

  return (
    <div className="w-full max-w-3xl mx-auto bg-zinc-950 border border-zinc-800 rounded-xl p-6 shadow-2xl font-mono text-sm">
      <div className="flex items-center justify-between pb-4 border-b border-zinc-800 mb-4">
        <div className="flex items-center space-x-2 text-zinc-400">
          <Terminal size={18} className="text-emerald-500" />
          <span>AgentSaaS Autonomous Shell</span>
        </div>
        <a
          href="https://your-gumroad-store.gumroad.com/l/agentsaas-pro"
          target="_blank"
          className="flex items-center gap-1 bg-gradient-to-r from-amber-500 to-orange-500 text-black px-3 py-1 rounded text-xs font-bold hover:opacity-90 transition-opacity"
        >
          <Zap size={14} /> Upgrade PRO (€29)
        </a>
      </div>

      <div className="h-64 overflow-y-auto mb-4 space-y-2 text-zinc-300">
        {logs.length === 0 ? (
          <p className="text-zinc-600">In attesa di istruzioni. Inserisci un obiettivo per l'agente...</p>
        ) : (
          logs.map((log, idx) => <p key={idx} className="leading-relaxed">{log}</p>)
        )}
      </div>

      <div className="flex gap-2">
        <input
          type="text"
          value={prompt}
          onChange={(e) => setPrompt(e.target.value)}
          placeholder="Es: Analizza i competitor e genera uno script di cold email..."
          className="flex-1 bg-zinc-900 border border-zinc-700 rounded px-4 py-2 text-zinc-200 focus:outline-none focus:border-emerald-500"
          disabled={loading}
        />
        <button
          onClick={handleRun}
          disabled={loading}
          className="bg-emerald-600 hover:bg-emerald-500 text-white px-5 py-2 rounded font-medium flex items-center gap-2 transition-colors disabled:opacity-50"
        >
          {loading ? 'Thinking...' : <><Send size={16} /> Run</>}
        </button>
      </div>
    </div>
  );
}
```

---

# 5. GROWTH & MONETIZATION PLAYBOOK (€0 ➔ €3.000/Mese)

### 📈 Funnel Architecture
```text
[GitHub Repo (MIT)] ── Traffic ──> [README + Terminal Upsell]
                                          │
                                          ▼
                             [Gumroad Checkout (€29)]
                                          │
                          ┌───────────────┴───────────────┐
                          ▼                               ▼
                 [Accesso Repo PRO]              [Community Discord]
```

### 1. GitHub Distribution Optimization
*   **Nome Repository:** Usa nomi orientati alla ricerca organica: `ai-agent-micro-saas-starter-kit` oppure `autonomous-agent-boilerplate`.
*   **Topics:** `fastapi`, `nextjs14`, `ai-agents`, `supabase`, `langchain`, `micro-saas`, `gumroad`, `indie-hacker`.
*   **Release Engineering:** Fai release v1.0.0, v1.1.0 costanti con changelog dettagliati; GitHub indicizza pesantemente i repository attivi.

### 2. Contenuti per l'acquisizione (X / Twitter & Reddit)
*   **Post Reddit su r/SaaS e r/SideProject:**
    *   *Titolo:* "Ho speso 2 mesi per standardizzare il codice dei miei 3 agenti AI monetizzati. Rilascio il core gratis su GitHub."
    *   *Contenuto:* Mostra i problemi reali risolti (webhook timeout, sync utente/crediti, gestione fallimenti dei tool). Linka la repo GitHub. Non linkare direttamente Gumroad su Reddit: lascia che sia il README del repository a convertire.
*   **X (Twitter) Build in Public:**
    *   Condividi video screen-recording di 30 secondi in cui esegui un comando nel terminale e fai vedere il credito scalato su Supabase e l'aggiornamento Stripe in tempo reale.

### 3. Copywriting della Gumroad Sales Page (€29)
*   **Headline:** *"Smetti di perdere 40 ore per ogni nuovo progetto AI. Lancia il tuo Micro-SaaS multi-agente oggi stesso."*
*   **Pain Points:** Spiega quanto costa comprare singolarmente template Next.js, configurare webhook Stripe da zero e implementare workflow multi-agente stabili.
*   **Value Stack dell'Upsell PRO:**
    1. Orchestrazione multi-agente ad alta resilienza (Supervisor + Worker + Critic).
    2. Moduli n8n pronti all'uso per generare lead in automatico.
    3. Template Next.js 14 completo con autenticazione via Supabase già integrata.
    4. Licenza commerciale senza restrizioni.
*   **Scarsità:** *Offerta di lancio a 29€ riservata ai primi 100 sviluppatori (prezzo finale: 79€).*

## 🔓 Get Full Commercial Access
👉 [Purchase License](https://py-forge.com)
