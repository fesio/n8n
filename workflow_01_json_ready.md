# 🚀 N8N WORKFLOW #1 - GOTOWY DO WGRANIA
## Workflow_01_DailyFeatureSelection - Kompletny JSON

Kopiuj cały poniższy JSON i wklej w n8n → "Import from JSON"

```json
{
  "name": "Workflow_01_DailyFeatureSelection",
  "nodes": [
    {
      "parameters": {
        "rule": {
          "interval": [
            "cron"
          ],
          "cronExpression": "0 3 * * *"
        }
      },
      "id": "uuid-cron-1",
      "name": "Cron",
      "type": "n8n-nodes-base.cron",
      "typeVersion": 1,
      "position": [
        50,
        100
      ],
      "disabled": false
    },
    {
      "parameters": {
        "mode": "each",
        "expression": "=[\n  {\"pair\": \"BTC/USDT\"},\n  {\"pair\": \"ETH/USDT\"},\n  {\"pair\": \"SOL/USDT\"}\n]"
      },
      "id": "uuid-batch-1",
      "name": "For Each Pair",
      "type": "n8n-nodes-base.splitInBatches",
      "typeVersion": 3,
      "position": [
        250,
        100
      ],
      "disabled": false
    },
    {
      "parameters": {
        "method": "GET",
        "url": "=https://api.github.com/repos/YOUR_USERNAME/trading-ai-system/contents/data/{{ $json.pair.replace('/', '_') }}_1h_latest.csv",
        "authentication": "predefinedCredentialType",
        "nodeCredentialType": "githubTokenApi",
        "headers": {
          "Accept": "application/vnd.github.v3.raw"
        }
      },
      "id": "uuid-http-get-1",
      "name": "HTTP GET - Read CSV",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.1,
      "position": [
        450,
        100
      ],
      "disabled": false
    },
    {
      "parameters": {
        "jsCode": "const csv = $json.body;\nconst lines = csv.split('\\n');\nconst headers = lines[0].split(',');\nconst data = [];\n\nfor (let i = 1; i < lines.length; i++) {\n  if (lines[i].trim()) {\n    const obj = {};\n    const values = lines[i].split(',');\n    headers.forEach((header, idx) => {\n      obj[header.trim()] = values[idx];\n    });\n    data.push(obj);\n  }\n}\n\nreturn { \n  data_parsed: data, \n  row_count: data.length,\n  pair: $json.pair \n};"
      },
      "id": "uuid-code-1",
      "name": "Code - Parse CSV",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        650,
        100
      ],
      "disabled": false
    },
    {
      "parameters": {
        "method": "POST",
        "url": "http://YOUR_SERVER:5000/api/calculate-indicators",
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            {
              "name": "csv_data",
              "value": "={{ $json.data_parsed }}"
            },
            {
              "name": "pair",
              "value": "={{ $json.pair }}"
            }
          ]
        }
      },
      "id": "uuid-http-post-1",
      "name": "HTTP POST - Python ML",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.1,
      "position": [
        850,
        100
      ],
      "disabled": false
    },
    {
      "parameters": {
        "resource": "repository",
        "operation": "createOrUpdateFile",
        "owner": "YOUR_USERNAME",
        "repository": "trading-ai-system",
        "filePath": "=/data/indicators_features_{{ $now.format('YYYY-MM-DD') }}_{{ $json.pair.replace('/', '_') }}.csv",
        "fileContent": "={{ $json.features_csv }}"
      },
      "id": "uuid-github-save-1",
      "name": "GitHub - Save CSV",
      "type": "n8n-nodes-base.github",
      "typeVersion": 1.1,
      "position": [
        1050,
        100
      ],
      "disabled": false
    },
    {
      "parameters": {
        "channel": "#strategy-dev",
        "text": "=✅ Features calculated for {{ $json.pair }}\n\nBars: {{ $json.row_count }}\nTimestamp: {{ $now.format('YYYY-MM-DD HH:mm:ss') }}\nIndicators: 30+ technical"
      },
      "id": "uuid-slack-1",
      "name": "Slack - Notify",
      "type": "n8n-nodes-base.slack",
      "typeVersion": 2,
      "position": [
        1250,
        100
      ],
      "disabled": false
    }
  ],
  "connections": {
    "Cron": {
      "main": [
        [
          {
            "node": "For Each Pair",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "For Each Pair": {
      "main": [
        [
          {
            "node": "HTTP GET - Read CSV",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "HTTP GET - Read CSV": {
      "main": [
        [
          {
            "node": "Code - Parse CSV",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Code - Parse CSV": {
      "main": [
        [
          {
            "node": "HTTP POST - Python ML",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "HTTP POST - Python ML": {
      "main": [
        [
          {
            "node": "GitHub - Save CSV",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "GitHub - Save CSV": {
      "main": [
        [
          {
            "node": "Slack - Notify",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

---

# 📥 KAK WGRAĆ:

1. **Kopiuj cały JSON wyżej** (od `{` do `}`)
2. **W n8n kliknij: Edit → Import from JSON**
3. **Wklej JSON**
4. **Kliknij Import**
5. **Zmień:**
   - `YOUR_USERNAME` → twoja nazwa GitHub
   - `YOUR_SERVER` → IP lub ngrok URL
6. **Save & Activate**

---

# 🔧 CO TRZEBA ZROBIĆ ZANIM BĘDZIE DZIAŁAĆ:

## KRYTYCZNE - WYKONAJ TEraz:

### 1. Utwórz GitHub Token

```
GitHub → Settings → Developer settings → Personal access tokens
→ Tokens (classic) → Generate new token (classic)
→ Scopes: ✓ repo (full control)
→ Copy token
```

### 2. Połącz GitHub w n8n

```
n8n → Credentials (top left) → Add New
→ Select: GitHub Token API
→ Paste token from step 1
→ Save
```

### 3. Połącz Slack Bot

```
Slack App: https://api.slack.com/apps
→ Create New App → From scratch
→ Name: n8n-trading
→ Workspace: twoja

→ OAuth & Permissions
→ Scopes: chat:write, channels:read
→ Install to workspace
→ Copy Bot Token
```

### 4. Połącz Slack w n8n

```
n8n → Credentials → Add New
→ Select: Slack
→ Paste Bot Token
→ Save
```

### 5. Uruchom Python Server

```bash
# Terminal:
cd scripts
pip install -r requirements.txt
python server.py

# Powinno wyświetlić:
# INFO:     Uvicorn running on http://0.0.0.0:5000
```

### 6. Jeśli Python lokalnie:

```bash
# W INNYM terminalu:
ngrok http 5000

# Kopiuj URL: https://xxxxx-xx-xxx.ngrok-free.app
# Wklej w JSON zamiast http://YOUR_SERVER:5000
```

---

# ✅ TEST WORKFLOW:

1. **Kliknij Play (Execute Workflow)**
2. **Monitor:**
   - Tab "Executions" - powinny być zielone checkmarks
   - Tab "Logs" - brak błędów
   - Slack - powinna przyjść wiadomość
   - GitHub - powinien być nowy plik CSV

---

# 🎯 Jeśli coś nie działa:

| Problem | Przyczyna | Rozwiązanie |
|---------|-----------|------------|
| "401 Unauthorized" | Zły GitHub token | Sprawdź scopes - musi być `repo` |
| "Connection refused" | Python server down | Uruchom: `python server.py` |
| "404 Not Found" | Zła ścieżka GitHub | Sprawdź `YOUR_USERNAME` |
| "No module found" | Brak bibliotek Python | `pip install -r requirements.txt` |
| Slack message nie przychodzi | Zły Bot token | Odłącz i podłącz znów w n8n |

---

# 🚀 NASTĘPNE KROKI:

Po 1️⃣ workflow'u działa → Zrób to samo dla:
- Workflow #2 (06:00 AM - Backtest)
- Workflow #3 (09:00 AM - Pine Generation)

---

**Powodzenia! 🎉**
