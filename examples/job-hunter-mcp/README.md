# Job Hunter MCP Brain — uruchomienie workflow n8n

Ten pakiet realizuje zasadę: **MCP planuje, dobiera trasę modelu i zatwierdza
zakres, Vercel Workflow zapewnia trwałość, n8n wykonuje integracje, PostgreSQL
przechowuje stan, Obsidian jest pamięcią, a człowiek podejmuje decyzję w
Telegramie**. Nie ma rankingu modeli ani zależności od NotebookLM Enterprise.
Workflow nie wysyła aplikacji automatycznie.

## Pliki

- `job_hunter_mcp_n8n_workflow.json` — workflow gotowy do importu w n8n.
- `job_hunter_mcp_system_postgresql.sql` — schemat PostgreSQL, prompty,
  deduplikacja, HITL, outbox i widoki operacyjne.
- `job_hunter_mcp_n8n_command.md` — gotowe polecenie wdrożeniowe i lista
  wartości wymaganych przed aktywacją.

## Wymagania

- n8n **2.37.10 lub nowszy**;
- PostgreSQL 14+ oraz prawo do instalacji rozszerzenia `pgcrypto`;
- MCP Brain wystawiony przez transport **Streamable HTTP**, z narzędziem
  `orchestrate_task`;
- credential OpenAI z dostępem do modelu obsługującego Responses API i Web
  Search; jednorazowo wybranym modelem dla wersji Trial jest `gpt-5-mini`;
- bot Telegram oraz własny Telegram Chat ID i User ID.

## 1. Zainstaluj bazę

```bash
psql "$DATABASE_URL" -v ON_ERROR_STOP=1 \
  -f job_hunter_mcp_system_postgresql.sql
```

Skrypt nie tworzy przykładowej osoby ani nie zapisuje sekretów. Dodaj prawdziwy
profil dopiero po uzyskaniu zgody na przetwarzanie:

```sql
WITH workspace AS (
  SELECT id
  FROM job_hunter.workspaces
  WHERE slug = 'default'
)
INSERT INTO job_hunter.candidate_profiles (
  workspace_id,
  full_name,
  email,
  phone,
  location,
  target_roles,
  work_modes,
  salary_expectation_min,
  salary_currency,
  raw_master_cv,
  profile_facts,
  consent_to_process,
  consent_recorded_at
)
SELECT
  workspace.id,
  'UZUPEŁNIJ IMIĘ I NAZWISKO',
  'UZUPEŁNIJ_EMAIL@example.com',
  NULL,
  'Polska / Remote',
  '["AI Workflow Architect", "Automation Engineer"]'::jsonb,
  '["remote", "hybrid"]'::jsonb,
  0,
  'PLN',
  'WKLEJ PRAWDZIWE MASTER CV — BEZ DOPISYWANIA FAKTÓW',
  '{}'::jsonb,
  true,
  CURRENT_TIMESTAMP
FROM workspace
RETURNING id;
```

Dodaj zweryfikowane umiejętności, używając zwróconego `candidate_id`:

```sql
INSERT INTO job_hunter.candidate_skills (
  candidate_id,
  normalized_name,
  category,
  proficiency_level,
  years_experience,
  evidence,
  verification_status
)
VALUES
  ('UZUPEŁNIJ_CANDIDATE_UUID', 'Python', 'language', 4, NULL,
   'Wskaż projekt lub stanowisko z CV', 'EVIDENCED'),
  ('UZUPEŁNIJ_CANDIDATE_UUID', 'n8n', 'automation', 4, NULL,
   'Wskaż konkretny workflow lub wdrożenie', 'EVIDENCED');
```

## 2. Wystaw MCP Brain

W węźle `Job Hunt Config` zamień wartość:

```text
SET_YOUR_PUBLIC_MCP_BRAIN_URL
```

na publiczny adres Streamable HTTP, na przykład
`https://twoj-serwer.example.com/mcp`. Klucz Bearer ustaw osobno w n8n
Credentials.

W lokalnym Docker Desktop przykładowy adres hosta to
`http://host.docker.internal:3001/mcp`, ale port i ścieżka muszą odpowiadać
faktycznie uruchomionemu serwerowi. Nie traktuj adresu przykładowego jako
działającego endpointu.

## 3. Zaimportuj workflow

W interfejsie n8n wybierz **Import from File** i wskaż
`job_hunter_mcp_n8n_workflow.json`. Możesz też użyć CLI:

```bash
n8n import:workflow --input=job_hunter_mcp_n8n_workflow.json
```

Workflow po imporcie pozostaje nieaktywny, aby nie wystartował przed
konfiguracją.

## 4. Podepnij Credentials

Poświadczenia są celowo pominięte w eksporcie. Wybierz je w n8n:

| Węzły                            | Credential                    |
| -------------------------------- | ----------------------------- |
| `MCP Orchestration Plan`         | HTTP Bearer Auth do MCP Brain |
| wszystkie węzły `Postgres`       | konto aplikacyjne PostgreSQL  |
| cztery węzły `OpenAI Chat Model` | OpenAI API                    |
| węzły i trigger `Telegram`       | Telegram Bot API              |

Tokeny, hasła i klucze trzymaj wyłącznie w n8n Credentials lub w menedżerze
sekretów.

## 5. Uzupełnij dwie wartości operatora

1. W `Job Hunt Config` zmień `SET_YOUR_PUBLIC_MCP_BRAIN_URL` na URL MCP.
2. W `Job Hunt Config` zmień `SET_YOUR_TELEGRAM_CHAT_ID` na Chat ID.
3. W `HITL Operator Config` zmień `SET_YOUR_TELEGRAM_USER_ID` na swój liczbowy
   User ID. Ten warunek blokuje decyzje innych użytkowników bota.

Opcjonalnie zmień w `Job Hunt Config` role, limit ofert, okno dni i próg
dopasowania. W ustawieniach trial domyślnie jest to 5 ofert, 7 dni i wynik co
najmniej 75/100.

Wywołanie MCP jawnie ustawia `knowledge_policy=adaptive_model_choice`,
`source_sensitivity=internal` i `durable_execution=true`. MCP wybiera z aktualnie
skonfigurowanych tras w chwili wykonania i zapisuje wybrany identyfikator modelu;
nie tworzy ani nie odczytuje rankingu. Materiały prywatne i tajne pozostają w
Obsidianie.

## 6. Role agentów i ich umiejętności

| Rola            | Umiejętność                                    | Twarda bramka                            |
| --------------- | ---------------------------------------------- | ---------------------------------------- |
| MCP Brain       | plan, zależności i dobór skonfigurowanej trasy | `controller=mcp_orchestrator`            |
| Model Router    | dobór modelu dla bieżącego zadania             | brak rankingu, zgodny kontrakt fallbacku |
| Vercel Workflow | checkpointy, retry i wznowienie planu MCP      | stały `idempotency_key`, maks. 3 próby   |
| Scout           | Web Search i ekstrakcja ofert do JSON          | URL, deduplikacja, skaner injection      |
| Evaluator       | ATS, gap analysis, mapa dowodów                | próg 75 i spełnione MUST_HAVE            |
| Tailor          | CV, list i e-mail wyłącznie z faktów           | identyfikatory dowodów                   |
| Critic          | niezależny audyt faktów i bezpieczeństwa       | `PASS`, zero niepopartych twierdzeń      |
| Człowiek        | approve / reject / changes w Telegramie        | jednorazowy token, 24 h                  |

Po decyzji Telegram workflow zmienia stan w bazie i dodaje zdarzenie do
`job_hunter.outbox_events`. Zatwierdzenie ustawia `automatic_submission_allowed`
na `false`; następny ruch należy do MCP. Konsument MCP/Obsidian powinien czytać
`job_hunter.v_outbox_ready`, przygotować zatwierdzony rekord pamięci i dopiero
wtedy zapisać go przez Obsidian bridge.

## 7. Test przed aktywacją

1. Uruchom `Manual Trial Start`.
2. Sprawdź kolejno: MCP plan → profil → Scout → zapis oferty → Evaluator.
3. Dla oferty z wynikiem co najmniej 75 sprawdź Tailor i Critic.
4. Kliknij przycisk Telegram. Oczekiwany wynik to zmiana stanu i wpis outbox,
   **bez wysłania aplikacji**.
5. Sprawdź stan:

```sql
SELECT * FROM job_hunter.v_application_pipeline ORDER BY overall_fit_score DESC;
SELECT * FROM job_hunter.v_pending_hitl_reviews;
SELECT * FROM job_hunter.v_outbox_ready;
```

Dopiero po przejściu testu aktywuj workflow. W produkcji ustaw retencję wykonań
n8n odpowiednią dla danych osobowych, użyj TLS dla MCP i ogranicz konto bazy do
niezbędnych uprawnień.

## Zakres wykonanych kontroli

- poprawny JSON i kompletna topologia 54 węzłów;
- pomyślny import przez CLI n8n 2.37.10;
- składnia 125 instrukcji skryptu sprawdzona parserem PostgreSQL;
- składnia 16 węzłów Code oraz 7 zapytań SQL workflow sprawdzona statycznie;
- brak osadzonych credentials i wzorców sekretów.

Pełny test end-to-end wymaga Twoich działających usług i Credentials; nie da się
go rzetelnie wykonać na pustych, celowo niezałączonych poświadczeniach.
