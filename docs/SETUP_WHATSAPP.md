# WhatsApp Cloud API — setup guide

This walks you from zero to a working WhatsApp webhook for Lumora. It has two
parts:

- **Part A — test it works (free, ~30 min):** message the bot yourself using
  Meta's free test number and see it reply.
- **Part B — go live to the pilot leads:** register your own number, get a
  template approved, and message the 20 leads.

By the end, the values you collect go into a `.env` file (copy from
[`.env.example`](../.env.example)). The code that consumes them lives in
`whatsapp/config.py`, and the webhook is served at `GET/POST /webhook` by
`api/main.py`.

---

## Glossary (the 5 values you need)

| `.env` variable | What it is | Where it comes from |
|---|---|---|
| `WHATSAPP_TOKEN` | Access token (Bearer) | API Setup page (temp) → System User (permanent) |
| `WHATSAPP_PHONE_NUMBER_ID` | ID of the *sending* number | API Setup page |
| `WHATSAPP_BUSINESS_ACCOUNT_ID` | WhatsApp Business Account (WABA) ID | API Setup page |
| `WHATSAPP_VERIFY_TOKEN` | A random string **you invent** | You make it up; reused in the webhook config |
| `WHATSAPP_APP_SECRET` | Signs inbound webhooks | App → Settings → Basic → App secret |

Note: the **Phone number ID** is NOT the phone number. It's a numeric ID shown
next to the number on the API Setup page.

---

## Part A — Test it works (free)

### 1. Create a Meta app

1. Go to <https://developers.facebook.com/apps> and log in.
2. **Create App** → choose use case **Other** → type **Business** → name it
   (e.g. "Lumora") → create.
3. On the app dashboard, find **WhatsApp** and click **Set up**.
   - If prompted, create or select a **Meta Business Account** to attach.

### 2. Grab the dev credentials

Open **WhatsApp → API Setup** in the left sidebar. On this page you'll see:

- A **test phone number** (Meta gives you one free) and its **Phone number ID**
  → copy into `WHATSAPP_PHONE_NUMBER_ID`.
- The **WhatsApp Business Account ID** → copy into `WHATSAPP_BUSINESS_ACCOUNT_ID`.
- A **temporary access token** (valid 24h — fine for testing) → copy into
  `WHATSAPP_TOKEN`. (We upgrade this to a permanent token in Part B, step 4.)

Then get the app secret:

- **App → Settings → Basic → App secret → Show** → copy into `WHATSAPP_APP_SECRET`.
  (You can leave this blank for purely local testing; signature checks are then
  skipped. Set it before anything is public.)

Finally, invent a verify token:

- Put any random string in `WHATSAPP_VERIFY_TOKEN` (e.g. run
  `python -c "import secrets; print(secrets.token_urlsafe(24))"`). You'll paste
  the *same* string into Meta's webhook config in step 5.

### 3. Fill in `.env`

```bash
cp .env.example .env
# edit .env and paste the 5 values
```

Confirm the app sees them:

```bash
.venv/Scripts/python.exe -c "from whatsapp.config import get_settings; s=get_settings(); print('configured?', s.is_configured())"
# -> configured? True
```

### 4. Run the API and expose it publicly

Meta must reach your webhook over **HTTPS**. For local dev, tunnel it.

```bash
# terminal 1 — run the API
.venv/Scripts/python.exe -m uvicorn api.main:app --reload --port 8000

# terminal 2 — expose it (pick one)
ngrok http 8000
#   or:  cloudflared tunnel --url http://localhost:8000
```

Copy the public HTTPS URL the tunnel prints, e.g.
`https://a1b2c3.ngrok-free.app`. Your webhook URL is that **+ `/webhook`**.

### 5. Configure the webhook in Meta

1. **WhatsApp → Configuration → Webhook → Edit**.
2. **Callback URL:** `https://<your-tunnel>/webhook`
3. **Verify token:** the exact string from `WHATSAPP_VERIFY_TOKEN`.
4. Click **Verify and save**. Meta calls `GET /webhook`; our endpoint echoes the
   challenge and the save succeeds. (If it fails, see Troubleshooting.)
5. Under **Webhook fields**, click **Manage** and **Subscribe** to **`messages`**.

### 6. Send yourself a message

1. On **API Setup**, under **To**, add your own phone number as an allowed
   recipient and confirm the code Meta sends you. (The free test number can only
   message numbers on this allow-list — up to 5.)
2. From your phone, send any text (e.g. `oi`) to the test number shown on the
   page.
3. Watch terminal 1. You should see the inbound logged and the bot reply with the
   welcome buttons from `whatsapp/flows/intake_pt_BR.json`.

If you get the welcome buttons back, **the whole pipeline works end to end.** 🎉

---

## Part B — Go live to the pilot leads

The free test number + 5-recipient limit is enough to prove the flow, but to
message the 20 pilot leads you need your own number and an approved template.

### 1. Add your own phone number

- **WhatsApp → API Setup → Add phone number** (or **WhatsApp Manager → Phone
  numbers**). Use a number **not currently active on the WhatsApp or WhatsApp
  Business app**.
- Set a **display name**, pick a category, and verify the number by SMS/call.
- Its **Phone number ID** replaces the test one in `WHATSAPP_PHONE_NUMBER_ID`.

### 2. Verify your business

- **Business Settings → Security Center → Start verification.**
- Required to lift messaging limits and to run marketing templates at volume.
- Allow a few days; Meta may ask for business documents.

### 3. Create and get a message template approved

First contact must use a **pre-approved template** (you can only send free-form
text within 24h of a user's last message — see the 24-hour window note below).

1. **WhatsApp Manager → Account tools → Message templates → Create template.**
2. Category **Marketing** (or **Utility** if it's transactional), language
   **Portuguese (BR) / `pt_BR`**.
3. Write the intro (keep it consistent with Sandra's flow). Example body:
   > Oi {{1}}! Aqui é a Lumora 💜 Ajudamos mulheres empreendedoras a conseguir
   > crédito justo usando o histórico do Pix. Posso te fazer algumas perguntas?
   - `{{1}}` is a variable our outreach fills with the lead's first name.
4. Submit and wait for **Approved** status (minutes to ~a day).

### 4. Mint a permanent access token

The temporary token expires in 24h. For anything ongoing, use a System User token:

1. **Business Settings → Users → System users → Add** → name it (e.g.
   "lumora-bot"), role **Admin** or **Employee**.
2. **Add assets** → assign your **app** (full control).
3. **Generate new token** → select the app → grant
   **`whatsapp_business_messaging`** and **`whatsapp_business_management`** →
   set expiry to **Never**.
4. Copy it into `WHATSAPP_TOKEN` (replaces the temporary one).

### 5. Preview, then send outreach

```bash
# Preview exactly who would be contacted — sends NOTHING:
.venv/Scripts/python.exe -m whatsapp.outreach --pilot 20 --dry-run

# Send for real, using your approved template name:
.venv/Scripts/python.exe -m whatsapp.outreach --pilot 20 --template lumora_intro --language pt_BR
```

The outreach script normalizes phone numbers, skips low-confidence ones by
default, logs every send to the SQLite store, and **refuses to run** if `.env`
isn't configured. Replies then flow into the webhook and the conversation engine
takes over automatically.

### 6. Watch the conversations

```bash
# contacts who asked for a human:
.venv/Scripts/python.exe -c "from whatsapp.store import get_store; [print(c['wa_id'], c['name']) for c in get_store().contacts_needing_agent()]"
```

Inbound media is saved under `data/media/<wa_id>/`; everything is queryable from
`data/messages.db`. Feed a finished conversation to the portfolio builder with
`portfolio.build_portfolio(wa_id, store)`.

---

## The 24-hour window (important)

WhatsApp only lets you send **free-form** messages within **24 hours** of the
user's last message to you. Outside that window you must send an **approved
template**. That's why:

- **First contact = template** (`whatsapp.outreach`).
- Once a lead replies, the bot can talk freely for 24h — which covers the whole
  intake flow. If someone goes quiet for >24h mid-flow, the next nudge has to be
  a template.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Webhook "Verify and save" fails | Tunnel not running, or `WHATSAPP_VERIFY_TOKEN` ≠ the token typed in Meta. Test directly: `curl "https://<tunnel>/webhook?hub.mode=subscribe&hub.verify_token=<your-token>&hub.challenge=42"` should print `42`. |
| `WhatsApp not configured: missing WHATSAPP_TOKEN...` | `.env` not loaded or values blank. Run the `is_configured()` check in Part A step 3. |
| No reply when you message the number | Not subscribed to the **messages** webhook field (Part A step 5), or your number isn't on the test allow-list (Part A step 6). |
| `(#131030) Recipient phone number not in allowed list` | Dev/test number can only message allow-listed recipients. Add them, or move to a live number (Part B). |
| Send fails outside 24h with a re-engagement error | You tried free-form text outside the 24h window — use a template. |
| Inbound webhooks rejected (403) | `WHATSAPP_APP_SECRET` is set but wrong, so signature validation fails. Re-copy it from App → Settings → Basic. |
| Token works then stops after a day | You're still on the temporary token. Mint a permanent System User token (Part B step 4). |
| `Graph API version` errors | Bump `WHATSAPP_GRAPH_API_VERSION` in `.env` to a current version (e.g. `v22.0`). |

---

## How this maps to the code

| Piece | File |
|---|---|
| Reads `.env` / exposes settings | `whatsapp/config.py` |
| `GET /webhook` verify, `POST /webhook` receive | `whatsapp/webhook.py` |
| Validates `X-Hub-Signature-256` | `whatsapp/signature.py` |
| Sends text/buttons/templates, downloads media | `whatsapp/client.py` |
| Persists messages, contacts, answers, media | `whatsapp/store.py` |
| Runs the conversation flow | `whatsapp/conversation.py` + `whatsapp/flows/intake_pt_BR.json` |
| First-contact outreach to leads | `whatsapp/outreach.py` |
