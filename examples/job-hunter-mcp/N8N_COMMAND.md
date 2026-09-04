# Polecenie wdrożeniowe — Job Hunter MCP dla n8n Trial

## Decyzja o modelu

W wersji Trial wybrano jeden model `gpt-5-mini` dla czterech agentów n8n.
Evaluator i Critic pracują z wysokim poziomem rozumowania, a Scout i Tailor z
poziomem średnim. MCP zachowuje prawo do wyboru logicznej trasy dla każdego
zadania, ale nie buduje ani nie przechowuje rankingu modeli.

## Instalacja z terminala

Uruchom w katalogu zawierającym trzy pliki pakietu:

```bash
psql "$DATABASE_URL" -v ON_ERROR_STOP=1 \
  -f job_hunter_mcp_system_postgresql.sql

n8n import:workflow \
  --input=job_hunter_mcp_n8n_workflow.json
```

Jeżeli n8n działa w kontenerze, skopiuj plik do kontenera i zaimportuj go:

```bash
docker cp job_hunter_mcp_n8n_workflow.json n8n:/tmp/job_hunter_mcp_n8n_workflow.json
docker exec n8n n8n import:workflow \
  --input=/tmp/job_hunter_mcp_n8n_workflow.json
```

Nazwa `n8n` musi odpowiadać faktycznej nazwie kontenera. Workflow po imporcie
pozostaje nieaktywny.

## Polecenie dla asystenta n8n

Wklej poniższe polecenie do asystenta n8n razem z plikiem JSON:

```text
Zaimportuj workflow „Job Hunter MCP Brain - n8n Trial + HITL” z załączonego
JSON. Nie zapisuj sekretów w węzłach. Podepnij istniejące Credentials:
OpenAI API, PostgreSQL, Telegram Bot API oraz HTTP Bearer Auth do MCP.
Zachowaj MCP jako controller=mcp_orchestrator, a n8n jako
n8n_role=optional_executor. Nie dodawaj rankingu modeli ani NotebookLM.
Pozostaw gpt-5-mini jako jednorazowo wybrany model Trial. Przed aktywacją
uruchom Manual Trial Start i zatrzymaj wykonanie przy pierwszym błędzie.
Nie włączaj automatycznego wysyłania aplikacji; wymagaj decyzji HITL.
```

## Kontrakt wywołania MCP

Węzeł `MCP Orchestration Plan` wywołuje `orchestrate_task` przez Streamable HTTP:

```json
{
  "objective": "Znajdź maksymalnie 5 aktualnych ofert zgodnych z profilem",
  "domain": "automation",
  "priority": "normal",
  "risk": "low",
  "needs_fresh_sources": true,
  "needs_external_integrations": true,
  "needs_schedule": true,
  "remember_result": true,
  "knowledge_policy": "adaptive_model_choice",
  "source_sensitivity": "internal",
  "durable_execution": true,
  "idempotency_key": "job-hunt-trial-YYYY-MM-DD"
}
```

Odpowiedź musi zawierać:

- `controller = mcp_orchestrator`;
- `n8n_role = optional_executor`;
- `model_route.strategy = runtime_capability_match`;
- krok `integration_execution` z `executor = n8n`;
- bramki kompletności, Critic i idempotency key.

## Wartości i Credentials wymagane przed testem

| Miejsce                | Wartość                                                 |
| ---------------------- | ------------------------------------------------------- |
| `Job Hunt Config`      | `SET_YOUR_PUBLIC_MCP_BRAIN_URL` → URL zakończony `/mcp` |
| `Job Hunt Config`      | `SET_YOUR_TELEGRAM_CHAT_ID` → docelowy Chat ID          |
| `HITL Operator Config` | `SET_YOUR_TELEGRAM_USER_ID` → dozwolony User ID         |
| n8n Credentials        | OpenAI API key                                          |
| n8n Credentials        | PostgreSQL URL/użytkownik/hasło z TLS                   |
| n8n Credentials        | Telegram Bot token                                      |
| n8n Credentials        | Bearer token serwera MCP                                |

Sekrety pozostają wyłącznie w n8n Credentials lub menedżerze sekretów. Nie
umieszczaj ich w eksporcie workflow, SQL, repozytorium ani Obsidianie.

## Test akceptacyjny

1. Wykonaj SQL z `ON_ERROR_STOP=1`.
2. Dodaj profil kandydata ze świadomą zgodą na przetwarzanie.
3. Podepnij cztery Credentials i trzy wartości operatora.
4. Uruchom `Manual Trial Start`.
5. Potwierdź kolejno: plan MCP, Scout, Evaluator, Tailor, Critic i Telegram HITL.
6. Sprawdź `job_hunter.v_outbox_ready`; zatwierdzenie nie może wysłać aplikacji
   automatycznie.
7. Aktywuj Schedule Trigger dopiero po pełnym teście manualnym.
