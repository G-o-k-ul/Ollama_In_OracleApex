# 🤖 Using OLLAMA in Oracle APEX

> Run AI models **completely offline** — no API keys, no usage costs, no data leaving your server — right inside your Oracle APEX application.

---

## What is OLLAMA?

[OLLAMA](https://ollama.com) is an open-source tool that lets you run large language models (LLMs) like Llama 3, Mistral, Gemma, and many others directly on your local machine — no cloud required. It exposes a simple REST API on `localhost:11434` that you can call just like any other web service.

| Feature | Details |
|---|---|
| 🔒 Privacy | Runs 100% offline — your data never leaves your machine |
| 🧠 Models | Supports dozens of open-source LLMs |
| 🔌 API | REST API compatible with the OpenAI format |
| 💰 Cost | Free and open-source (MIT license) |

---

## Architecture Overview

```
Oracle APEX Page
      │
      ▼
PL/SQL Process (APEX_WEB_SERVICE.MAKE_REST_REQUEST)
      │
      ▼
OLLAMA REST API  ←─  http://localhost:11434/api/chat
      │
      ▼
Local LLM Model (llama3, mistral, gemma, etc.)
      │
      ▼
JSON Response  ──►  Parsed & Displayed in APEX
```

---

## Step 1 — Install OLLAMA on Your Server

Visit [ollama.com](https://ollama.com) and download for your OS.

**On Linux:**
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

**Start the service:**
```bash
ollama serve
```

**Pull a model** (Llama 3 is recommended for general use):
```bash
ollama pull llama3
```

Other popular models:
```bash
ollama pull mistral
ollama pull gemma:2b
ollama pull phi3
```

> **Note:** If OLLAMA is on the same machine as Oracle DB, use `localhost`. If it's on a separate server, use its IP and open port `11434` in the firewall.

---

## Step 2 — Expose OLLAMA on a Network-Accessible Address

By default, OLLAMA only listens on `127.0.0.1`. To bind it to all interfaces:

```bash
OLLAMA_HOST=0.0.0.0 ollama serve
```

To set this permanently via the systemd service file:

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0"
```

Then reload:
```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

---

## Step 3 — Grant Oracle DB Access via ACL

Oracle uses Access Control Lists (ACLs) to restrict outbound HTTP connections. Run the following as `SYS` or a DBA account:

```sql
BEGIN
  DBMS_NETWORK_ACL_ADMIN.APPEND_HOST_ACE(
    host        => '192.168.1.100',   -- IP of your OLLAMA server
    lower_port  => 11434,
    upper_port  => 11434,
    ace         => xs$ace_type(
                     privilege_list => xs$name_list('connect'),
                     principal_name => 'APEX_PUBLIC_USER',
                     principal_type => xs_acl.ptype_db
                   )
  );
  COMMIT;
END;
/
```

> ⚠️ Replace `192.168.1.100` with your actual OLLAMA server IP.  
> ⚠️ Replace `APEX_PUBLIC_USER` with your workspace schema if different.

---

## Step 4 — Test OLLAMA from the Command Line

Before touching APEX, verify OLLAMA works with a direct `curl` call:

```bash
curl http://localhost:11434/api/chat \
  -d '{
    "model": "llama3",
    "messages": [
      { "role": "user", "content": "What is Oracle APEX?" }
    ],
    "stream": false
  }'
```

**Expected response:**
```json
{
  "model": "llama3",
  "response": "Oracle APEX (Application Express) is a low-code development platform...",
  "done": true
}
```

---

## Step 5 — Call OLLAMA from Oracle APEX (PL/SQL Process)

In Page Designer, go to **Processing** and add a new **PL/SQL Process**:

```sql
DECLARE
  l_url       VARCHAR2(500)  := 'http://192.168.1.100:11434/api/chat';
  l_body      CLOB;
  l_response  CLOB;
  l_json      APEX_JSON.t_values;
  l_answer    CLOB;
BEGIN
  -- Build the JSON request body
  APEX_JSON.INITIALIZE_CLOB_OUTPUT;
  APEX_JSON.OPEN_OBJECT;
    APEX_JSON.WRITE('model',  'llama3');
    APEX_JSON.WRITE('stream', FALSE);
    APEX_JSON.OPEN_ARRAY('messages');
      APEX_JSON.OPEN_OBJECT;
        APEX_JSON.WRITE('role',    'user');
        APEX_JSON.WRITE('content', :P1_USER_QUESTION);
      APEX_JSON.CLOSE_OBJECT;
    APEX_JSON.CLOSE_ARRAY;
  APEX_JSON.CLOSE_OBJECT;
  l_body := APEX_JSON.GET_CLOB_OUTPUT;
  APEX_JSON.FREE_OUTPUT;

  -- Make the REST call to OLLAMA
  l_response := APEX_WEB_SERVICE.MAKE_REST_REQUEST(
    p_url         => l_url,
    p_http_method => 'POST',
    p_body        => l_body,
    p_wallet_path => NULL
  );

  -- Parse the JSON response
  APEX_JSON.PARSE(l_json, l_response);
  l_answer := APEX_JSON.GET_CLOB_VALUE(
    p_path   => 'response',
    p_values => l_json
  );

  -- Push result to page item
  :P1_AI_RESPONSE := l_answer;

EXCEPTION
  WHEN OTHERS THEN
    :P1_AI_RESPONSE := 'Error: ' || SQLERRM;
END;
```

---

## Step 6 — Build the APEX Page UI

Your page needs three components:

| Component | Name | Notes |
|---|---|---|
| Text Area | `P1_USER_QUESTION` | User types their prompt here |
| Button | `Ask AI` | Action = Submit Page |
| Text Area | `P1_AI_RESPONSE` | Set to **Read Only** to display the AI's answer |

### Non-Blocking Option (AJAX Callback)

For a smoother experience without a full page reload, use a **Dynamic Action** with an AJAX Callback:

```javascript
// JavaScript on button click (Dynamic Action → Execute JavaScript Code)
apex.server.process('CALL_OLLAMA', {
  pageItems: '#P1_USER_QUESTION'
}, {
  success: function(data) {
    apex.item('P1_AI_RESPONSE').setValue(data.answer);
  }
});
```

---

## Step 7 — OpenAI-Compatible API (Optional)

OLLAMA also supports the OpenAI `/v1/chat/completions` endpoint format — useful if you're already familiar with that structure:

```bash
curl http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3",
    "messages": [
      { "role": "user", "content": "Explain PL/SQL cursors in simple terms" }
    ]
  }'
```

Parse the response at `choices[0].message.content` using `APEX_JSON`.

---

## Practical Use Cases

- 📦 Auto-generate shipment remarks or GL narrations from structured data
- 📧 Summarize long customer email threads inside an APEX page
- 🔍 Let users query data in plain English and convert it to SQL
- 🤖 Internal FAQ bot powered by your own documents (RAG)
- 📒 Draft journal entry descriptions from accounting transactions

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `ORA-29273` (HTTP request failed) | ACL not configured | Revisit [Step 3](#step-3--grant-oracle-db-access-via-acl) |
| `Connection refused` | OLLAMA not running or bound to `localhost` only | Revisit [Step 2](#step-2--expose-ollama-on-a-network-accessible-address) |
| Slow first response | Model loads into RAM on first call | Normal — subsequent calls are much faster |
| Empty response | `"stream": false` missing from request body | Ensure it's included in your JSON payload |

---

## License

OLLAMA is free and open-source under the [MIT License](https://github.com/ollama/ollama/blob/main/LICENSE).
