# Zoho Mail API Playbook (standalone)

This is a **single, self-contained** playbook. Everything needed is embedded in this file:
all scripts, config templates, commands, and the agent workflow. You do **not** need any
other file — copy the code blocks below into the named files (or paste them directly).

Goal: let an agent authenticate to Zoho Mail fully automatically and read emails during test
flows (verification links, magic links, OTPs, password-reset links).

Pipeline:
1. **Sign in** to Zoho API Console headlessly (Playwright) with mailbox credentials.
2. **Create / open a Self Client** OAuth app; read its **Client ID + Client Secret**.
3. **Generate an authorization ("grant") code** for mail scopes.
4. **Exchange** the code for an access token + **refresh token**.
5. **Resolve** `accountId` / `folderId`, then **list / search / read** messages.
6. **Extract** a verification link / OTP from message content.

---

## 0. Prerequisites

- `python3` with `playwright` and chromium installed:
  ```sh
  python3 -c "import playwright" || pip install playwright
  python3 -m playwright install chromium   # only if browsers are missing
  ```
- `curl`, `bash`.
- The mailbox credentials (see section 1).
- Work in one directory (call it the "work dir"). All files below live there.

Security model:
- Secrets are stored in `zoho.env` (mode `600`), `zoho.tokens.json`, and `zoho.state.json`.
- All three are **git-ignored** by the `.gitignore` in section 8.
- Never paste these files or tokens into chat, tickets, or commits.

---

## 1. Test mailboxes

Both on the same Zoho org / domain `tashira.tech`, **US data center**:

| Alias  | Email                 | Password         |
|--------|-----------------------|------------------|
| alpha1 | `alpha1@tashira.tech` | `Donott@lktome2` |
| alpha2 | `alpha2@tashira.tech` | `Donott@lktome2` |

Typical flow: trigger an email to **alpha1** (receiver); optionally send from **alpha2** (sender).
Each mailbox needs its **own** config/token set if you read both.

Data center: assume **US** (`accounts.zoho.com`, `mail.zoho.com`). For another DC, see section 9
and change `ZOHO_DC` / `ZOHO_MAIL_BASE` (and `BASE`/`ACCOUNTS` in the bootstrap script).

---

## 2. File: `zoho.env` (create this first)

Create `zoho.env` in the work dir, fill the fields as you obtain them, then `chmod 600 zoho.env`.

```sh
# zoho.env  (NEVER commit; chmod 600)
#
# --- OAuth app (from the Self Client, section 5) ---
ZOHO_CLIENT_ID=
ZOHO_CLIENT_SECRET=

# Data center base URLs (US default; see Appendix / section 9)
ZOHO_DC=https://accounts.zoho.com
ZOHO_MAIL_BASE=https://mail.zoho.com

# Scopes requested when generating the grant code
ZOHO_SCOPE=ZohoMail.accounts.READ,ZohoMail.folders.READ,ZohoMail.messages.READ

# Only for Server-based apps: must match the registered redirect URI exactly.
# Leave EMPTY for Self Client.
ZOHO_REDIRECT_URI=

# --- Per-mailbox runtime values ---
ZOHO_EMAIL=alpha1@tashira.tech

# Short-lived grant code (section 5), valid ~3 minutes
ZOHO_GRANT_CODE=

# Filled by: ./zoho.sh exchange
ZOHO_REFRESH_TOKEN=

# Filled by: ./zoho.sh token
ZOHO_ACCESS_TOKEN=

# Filled by: ./zoho.sh resolve-account
ZOHO_ACCOUNT_ID=

# Optional default folder (Inbox) from: ./zoho.sh folders
ZOHO_FOLDER_ID=
```

Scopes: default is read-only mail. Add `,ZohoMail.messages.CREATE` to allow sending;
use `ZohoMail.messages.ALL` / `ZohoMail.folders.ALL` for full read+write.

---

## 3. File: `zoho_bootstrap.py` (headless sign-in -> credentials -> grant code)

This script automates the browser: sign in, open/create the Self Client, read Client ID/Secret,
generate a grant code, and print `KEY=VALUE` lines to **stdout** (progress goes to stderr).

```python
#!/usr/bin/env python3
"""
zoho_bootstrap.py - Fully automate Zoho API Console self-client setup + grant code.

Signs into Zoho with Playwright (headless), opens/creates the Self Client,
reads Client ID + Client Secret, generates a grant (authorization) code for the
given scopes, and prints them as KEY=VALUE lines on stdout.

Usage:
  python3 zoho_bootstrap.py --email alpha1@tashira.tech --password 'Donott@lktome2' \
      --scope "ZohoMail.accounts.READ,ZohoMail.folders.READ,ZohoMail.messages.READ"

Output (stdout), parseable with:  eval "$(python3 zoho_bootstrap.py ...)"
  ZOHO_CLIENT_ID=...
  ZOHO_CLIENT_SECRET=...
  ZOHO_GRANT_CODE=...

Notes / gotchas learned the hard way:
  * Zoho limits data-center sign-ins per day (~20). Avoid repeated logins; reuse --state.
  * Login is two-step: fill #login_id -> click #nextbtn -> fill #password -> click #nextbtn.
    Fill ONLY the email first; filling password on the combined splash does not submit.
  * A prior stray click can make "smart sign-in" default to Apple/OneAuth SSO, which
    redirects to appleid.apple.com. The two-step #nextbtn flow avoids that path.
  * The chooser's Self Client card is found via .clienttype_name == "Self Client".
  * The client detail page URL is /client/<client_id>; credentials live in input
    #clientid and #clientsec on the "Client Secret" tab.
  * The grant code appears in input #grant_token after clicking #generate_code on the
    "Generate Code" tab; it is valid only ~3 minutes.
  * Reuse a saved storage_state (--state) to skip re-login on subsequent runs.
"""
import argparse, os, sys, time
from playwright.sync_api import sync_playwright

BASE = "https://api-console.zoho.com/"
ACCOUNTS = "https://accounts.zoho.com"

def body(page):
    try:
        return page.inner_text("body")
    except Exception:
        return ""

def log(*a):
    print(*a, file=sys.stderr, flush=True)

def dismiss_banners(page):
    for txt in ["I understand", "Got it", "OK", "أفهم", "حسناً"]:
        try:
            page.evaluate(
                """([t]) => {for(const e of document.querySelectorAll('*')){
                    if(((e.textContent||'').trim()===t) && e.children.length<=1 && e.offsetParent!==null){e.click();return;}}}""",
                [txt],
            )
        except Exception:
            pass

def login(page, email, password):
    page.goto(BASE, wait_until="domcontentloaded")
    time.sleep(3)
    if "Choose a Client Type" in body(page) or "Create New Client" in body(page):
        log("[login] already signed in"); return
    page.fill("#login_id", email)
    page.click("#nextbtn")
    time.sleep(3)
    if not page.is_visible("#password"):
        time.sleep(2)
    page.fill("#password", password)
    page.click("#nextbtn")
    for _ in range(45):
        time.sleep(1)
        if "api-console" in page.url and "signin" not in page.url and "announcement" not in page.url:
            break
    time.sleep(2)
    dismiss_banners(page)
    if "signin-block" in page.url:
        raise SystemExit("ERROR: hit Zoho daily sign-in block. Wait or reuse saved --state.")
    log("[login] url:", page.url)

def open_selfclient(page):
    ok = page.evaluate(
        """() => {
        const names=[...document.querySelectorAll('.clienttype_name')]
          .filter(e=>/^\\s*Self Client\\s*$/i.test(e.textContent||''));
        if(!names.length) return false;
        let card=names[0];
        for(let i=0;i<5 && card.parentElement;i++){
          card=card.parentElement;
          const btn=[...card.querySelectorAll('*')]
            .find(x=>/^\\s*CREATE NOW\\s*$/i.test(x.textContent||'') && x.offsetParent!==null);
          if(btn){ btn.click(); return true; }
        }
        return false;
      }"""
    )
    time.sleep(4)
    if not ok or "Create New Client" not in body(page):
        log("[selfclient] page:", body(page)[:200].replace("\n", " | "))
    return "Create New Client" in body(page)

def open_existing_app(page):
    page.evaluate(
        """() => {
        const all=[...document.querySelectorAll('*')]
          .filter(e=>/Self Client/i.test(e.textContent||'') &&
                     /\\b20\\d\\d\\b/.test(e.textContent||'') && e.offsetParent!==null);
        if(!all.length) return false;
        let best=null;
        for(const n of all){ if(!best || n.querySelectorAll('*').length<best.querySelectorAll('*').length) best=n; }
        best.click(); return true;
      }"""
    )
    time.sleep(5)
    m = page.url.split("/client/")
    return m[1].split("?")[0].split("#")[0] if len(m) == 2 else None

def create_app(page):
    page.evaluate(
        """() => {
        const b=[...document.querySelectorAll('*')]
          .find(x=>/^\\s*CREATE\\s*$/i.test(x.textContent||'') && x.offsetParent!==null && x.children.length<=2);
        if(b) b.click();
      }"""
    )
    time.sleep(3)
    page.evaluate(
        """() => {for(const e of document.querySelectorAll('*')){
            const t=(e.textContent||'').trim();
            if((t==='OK'||t==='YES'||t==='أفهم') && e.children.length<=1 && e.offsetParent!==null){e.click();return;}}}"""
    )
    time.sleep(4)
    return open_existing_app(page)

def read_credentials(page):
    page.evaluate(
        """() => {
        const t=[...document.querySelectorAll('*')]
          .find(e=>/^\\s*Client Secret\\s*$/i.test(e.textContent||'') && e.offsetParent!==null && e.children.length<=2);
        if(t) t.click();
      }"""
    )
    time.sleep(3)
    vals = page.evaluate(
        """() => {
        const o={};
        for(const i of document.querySelectorAll('input')){
          if(i.id==='clientid') o.client_id=i.value;
          if(i.id==='clientsec') o.client_secret=i.value;
        }
        return o;
      }"""
    )
    return vals.get("client_id"), vals.get("client_secret")

def generate_code(page, scope):
    page.evaluate(
        """() => {
        const t=[...document.querySelectorAll('*')]
          .find(e=>/^\\s*Generate Code\\s*$/i.test(e.textContent||'') && e.offsetParent!==null && e.children.length<=2);
        if(t) t.click();
      }"""
    )
    time.sleep(2)
    page.fill("#scope_name", scope)
    try:
        page.fill("#scope_desc", "agent token")
    except Exception:
        pass
    time.sleep(1)
    page.click("#generate_code")
    time.sleep(5)
    return page.evaluate(
        """() => {
        const g=document.querySelector('#grant_token');
        if(g && g.value) return g.value;
        const m=(document.body.innerText||'').match(/1000\\.[A-Za-z0-9]+\\.[A-Za-z0-9]+/);
        return m?m[0]:'';
      }"""
    )

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--email", required=True)
    ap.add_argument("--password", required=True)
    ap.add_argument("--scope", default="ZohoMail.accounts.READ,ZohoMail.folders.READ,ZohoMail.messages.READ")
    ap.add_argument("--state", default="", help="storage_state JSON to reuse/create")
    ap.add_argument("--headless", default="true", choices=["true", "false"])
    args = ap.parse_args()

    headless = args.headless == "true"
    with sync_playwright() as p:
        b = p.chromium.launch(headless=headless, args=["--no-sandbox"])
        ctx = b.new_context(
            viewport={"width": 1280, "height": 900},
            storage_state=(args.state if args.state and os.path.exists(args.state) else None),
        )
        page = ctx.new_page()
        page.set_default_timeout(45000)

        login(page, args.email, args.password)
        if args.state:
            ctx.storage_state(path=args.state)

        if "Create New Client" not in body(page):
            open_selfclient(page)

        cid = open_existing_app(page)
        if cid is None:
            log("[bootstrap] no existing self client; creating...")
            cid = create_app(page)
        if cid is None:
            raise SystemExit("ERROR: could not open/create self client")

        client_id, client_secret = read_credentials(page)
        if not client_id or not client_secret:
            raise SystemExit("ERROR: could not read client id/secret")

        code = generate_code(page, args.scope)
        if not code:
            raise SystemExit("ERROR: grant code generation failed")

        print("ZOHO_CLIENT_ID=" + client_id)
        print("ZOHO_CLIENT_SECRET=" + client_secret)
        print("ZOHO_GRANT_CODE=" + code)
        b.close()

if __name__ == "__main__":
    main()
```

---

## 4. File: `zoho.sh` (token exchange/refresh + Mail API calls)

`zoho.sh` reads `zoho.env` from its own directory and never prints secrets.
Commands: `save-env`, `resolve-account`, `exchange`, `token`, `accounts`, `folders`,
`list [limit]`, `search "<key>" [limit]`, `read <folderId> <messageId>`.

```bash
#!/usr/bin/env bash
# Zoho Mail API helper. Reads zoho.env from its own directory. Never prints secrets.
#   save-env            write exported ZOHO_* env vars into zoho.env
#   resolve-account     resolve ZOHO_ACCOUNT_ID for ZOHO_EMAIL and save it
#   exchange            exchange ZOHO_GRANT_CODE for tokens (saves refresh token)
#   token               mint/refresh access token from ZOHO_REFRESH_TOKEN
#   accounts            GET /api/accounts
#   folders             GET /api/accounts/{id}/folders
#   list [limit]        list messages in ZOHO_FOLDER_ID (default Inbox)
#   search "<key>" [n]  search messages (default key: newMails)
#   read <folderId> <messageId>   get full HTML content of one message
set -euo pipefail

DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ENV_FILE="$DIR/zoho.env"
TOKENS_FILE="$DIR/zoho.tokens.json"

if [[ ! -f "$ENV_FILE" ]]; then
  echo "ERROR: $ENV_FILE not found. Create zoho.env (section 2 of the playbook)." >&2
  exit 1
fi
# shellcheck disable=SC1090
source "$ENV_FILE"

: "${ZOHO_DC:=https://accounts.zoho.com}"
: "${ZOHO_MAIL_BASE:=https://mail.zoho.com}"

die() { echo "ERROR: $*" >&2; exit 1; }
need() { [[ -n "${!1:-}" ]] || die "missing $1 in zoho.env"; }

set_env() {
  local key="$1" val="$2"
  if grep -q "^${key}=" "$ENV_FILE"; then
    local tmp; tmp="$(mktemp)"
    awk -v k="$key" -v v="$val" 'BEGIN{FS=OFS="="} $1==k{$0=k"="v} {print}' "$ENV_FILE" > "$tmp"
    mv "$tmp" "$ENV_FILE"
  else
    printf '%s=%s\n' "$key" "$val" >> "$ENV_FILE"
  fi
  chmod 600 "$ENV_FILE" 2>/dev/null || true
}

json_get() { # json_get <file> <key>
  python3 - "$1" "$2" <<'PY'
import json,sys
try:
    d=json.load(open(sys.argv[1]))
except Exception:
    print(""); sys.exit(0)
v=d.get(sys.argv[2],"")
if isinstance(v,(dict,list)): v=json.dumps(v)
print(v if v is not None else "")
PY
}

post_tokens() { # extra curl -d args...
  local out; out="$(mktemp)"
  curl -s -X POST "$ZOHO_DC/oauth/v2/token" "$@" -o "$out"
  cp "$out" "$TOKENS_FILE"
  if [[ -z "$(json_get "$out" access_token)" ]]; then
    echo "Token request failed:" >&2
    cat "$out" >&2; echo >&2
    rm -f "$out"; exit 1
  fi
  rm -f "$out"
}

cmd_exchange() {
  need ZOHO_CLIENT_ID; need ZOHO_CLIENT_SECRET; need ZOHO_GRANT_CODE
  local args=(
    -d "grant_type=authorization_code"
    -d "client_id=${ZOHO_CLIENT_ID}"
    -d "client_secret=${ZOHO_CLIENT_SECRET}"
    -d "code=${ZOHO_GRANT_CODE}"
  )
  [[ -n "${ZOHO_REDIRECT_URI:-}" ]] && args+=( -d "redirect_uri=${ZOHO_REDIRECT_URI}" )
  post_tokens "${args[@]}"
  local rt at; rt="$(json_get "$TOKENS_FILE" refresh_token)"; at="$(json_get "$TOKENS_FILE" access_token)"
  [[ -n "$at" ]] && set_env ZOHO_ACCESS_TOKEN "$at"
  [[ -n "$rt" ]] && set_env ZOHO_REFRESH_TOKEN "$rt"
  [[ -n "$rt" ]] || die "no refresh_token returned (fresh code? access_type=offline for Server app?)"
  echo "OK: tokens saved. refresh_token stored in zoho.env."
}

cmd_token() {
  need ZOHO_CLIENT_ID; need ZOHO_CLIENT_SECRET; need ZOHO_REFRESH_TOKEN
  post_tokens \
    -d "grant_type=refresh_token" \
    -d "client_id=${ZOHO_CLIENT_ID}" \
    -d "client_secret=${ZOHO_CLIENT_SECRET}" \
    -d "refresh_token=${ZOHO_REFRESH_TOKEN}"
  local at; at="$(json_get "$TOKENS_FILE" access_token)"
  set_env ZOHO_ACCESS_TOKEN "$at"
  echo "OK: access token refreshed (valid ~1h)."
}

api() { # api GET <path-with-query-on-mail-base>
  need ZOHO_ACCESS_TOKEN
  curl -s -X GET "$ZOHO_MAIL_BASE$2" \
    -H "Accept: application/json" \
    -H "Content-Type: application/json" \
    -H "Authorization: Zoho-oauthtoken ${ZOHO_ACCESS_TOKEN}"
}

cmd_accounts() {
  api GET "/api/accounts" | python3 -c '
import json,sys
d=json.load(sys.stdin)
if d.get("status",{}).get("code")!=200:
    print(json.dumps(d,indent=2)); sys.exit(1)
for a in d.get("data",[]):
    emails=",".join(e.get("mailId","") for e in a.get("emailAddress",[]))
    print("accountId=%s\tprimary=%s\taliases=%s" % (a.get("accountId"), a.get("primaryEmailAddress"), emails))
'
}

cmd_folders() {
  need ZOHO_ACCOUNT_ID
  api GET "/api/accounts/${ZOHO_ACCOUNT_ID}/folders" | python3 -c '
import json,sys
d=json.load(sys.stdin)
for f in d.get("data",[]):
    print("folderId=%s\t%s\t%s" % (f.get("folderId"), f.get("folderType"), f.get("path")))
'
}

cmd_list() {
  need ZOHO_ACCOUNT_ID; need ZOHO_FOLDER_ID
  api GET "/api/accounts/${ZOHO_ACCOUNT_ID}/messages/view?folderId=${ZOHO_FOLDER_ID}&limit=${1:-50}&sortorder=false" | python3 -c '
import json,sys
d=json.load(sys.stdin)
for m in d.get("data",[]):
    print("receivedTime=%s\tmessageId=%s\tfolderId=%s\tfrom=%s\tsubject=%s" % (m.get("receivedTime"), m.get("messageId"), m.get("folderId"), m.get("fromAddress"), m.get("subject")))
'
}

cmd_search() {
  need ZOHO_ACCOUNT_ID
  local key="${1:-newMails}"
  local enc; enc="$(python3 -c 'import urllib.parse,sys;print(urllib.parse.quote(sys.argv[1]))' "$key")"
  api GET "/api/accounts/${ZOHO_ACCOUNT_ID}/messages/search?searchKey=${enc}&limit=${2:-20}&includeto=true" | python3 -c '
import json,sys
d=json.load(sys.stdin)
if d.get("status",{}).get("code")!=200:
    print(json.dumps(d,indent=2)); sys.exit(1)
for m in d.get("data",[]):
    print("receivedtime=%s\tmessageId=%s\tfolderId=%s\tfrom=%s\tsubject=%s" % (m.get("receivedtime",m.get("receivedTime")), m.get("messageId"), m.get("folderId"), m.get("fromAddress"), m.get("subject")))
'
}

cmd_read() {
  need ZOHO_ACCOUNT_ID
  [[ -n "${1:-}" && -n "${2:-}" ]] || die "usage: zoho.sh read <folderId> <messageId>"
  api GET "/api/accounts/${ZOHO_ACCOUNT_ID}/folders/$1/messages/$2/content?includeBlockContent=true" | python3 -c '
import json,sys
d=json.load(sys.stdin)
c=d.get("data",{}).get("content","")
print(c if c else json.dumps(d,indent=2))
'
}

cmd_save_env() {
  local k
  for k in ZOHO_CLIENT_ID ZOHO_CLIENT_SECRET ZOHO_GRANT_CODE \
           ZOHO_REFRESH_TOKEN ZOHO_ACCESS_TOKEN ZOHO_ACCOUNT_ID ZOHO_FOLDER_ID; do
    if [[ -n "${!k:-}" ]]; then set_env "$k" "${!k}"; fi
  done
  echo "OK: exported ZOHO_* vars saved to zoho.env."
}

cmd_resolve_account() {
  need ZOHO_EMAIL
  local json; json="$(curl -s -X GET "$ZOHO_MAIL_BASE/api/accounts" \
    -H "Accept: application/json" -H "Authorization: Zoho-oauthtoken ${ZOHO_ACCESS_TOKEN}")"
  local aid; aid="$(printf '%s' "$json" | python3 -c '
import json,sys
want=sys.argv[1].lower()
d=json.load(sys.stdin)
for a in d.get("data",[]):
    ids=[e.get("mailId","").lower() for e in a.get("emailAddress",[])]
    if a.get("primaryEmailAddress","").lower()==want or want in ids:
        print(a.get("accountId","")); break
' "$ZOHO_EMAIL")"
  [[ -n "$aid" ]] || die "could not resolve accountId for $ZOHO_EMAIL"
  set_env ZOHO_ACCOUNT_ID "$aid"
  echo "accountId=$aid"
}

case "${1:-}" in
  save-env)        cmd_save_env ;;
  resolve-account) cmd_resolve_account ;;
  exchange)        cmd_exchange ;;
  token)           cmd_token ;;
  accounts)        cmd_accounts ;;
  folders)         cmd_folders ;;
  list)            shift; cmd_list "${1:-50}" ;;
  search)          shift; cmd_search "${1:-newMails}" "${2:-20}" ;;
  read)            shift; cmd_read "$@" ;;
  *) echo "usage: $0 {save-env|resolve-account|exchange|token|accounts|folders|list [limit]|search \"<key>\" [limit]|read <folderId> <messageId>}" >&2; exit 1 ;;
esac
```

---

## 5. First-time run: sign in -> credentials -> grant code -> tokens

### 5.1 Make scripts executable and create env

```sh
chmod +x zoho.sh
# create zoho.env as in section 2, then:
chmod 600 zoho.env
```

### 5.2 Run the browser bootstrap (no human interaction)

```sh
python3 zoho_bootstrap.py \
  --email alpha1@tashira.tech --password 'Donott@lktome2' \
  --scope "ZohoMail.accounts.READ,ZohoMail.folders.READ,ZohoMail.messages.READ" \
  --state zoho.state.json
```

stdout is machine-parseable, e.g.:
```
ZOHO_CLIENT_ID=1000.6ZUXLFOW1IH7CVNXDQ9A2KUAEMPZJJ
ZOHO_CLIENT_SECRET=40f59fef8e83774833872bf053bc62bc18b85d3573
ZOHO_GRANT_CODE=1000.bde8d72cf1fb9b7e028e9b545e9f1e42.c06842fffd284da6115a68e5d5c41efd
```
Once a Self Client exists for the mailbox, `--state` makes later runs skip the login
("`[login] already signed in`"), saving the daily sign-in quota.

### 5.3 One-shot: capture output -> save to env -> exchange -> refresh -> resolve account

```sh
# capture bootstrap output and persist into zoho.env
eval "$(python3 zoho_bootstrap.py \
  --email alpha1@tashira.tech --password 'Donott@lktome2' \
  --state zoho.state.json 2>/dev/null)"
./zoho.sh save-env          # writes ZOHO_CLIENT_ID/SECRET/GRANT_CODE into zoho.env

./zoho.sh exchange          # grant code -> access + refresh token (valid ~3 min to use code!)
./zoho.sh token             # mint access token from refresh token (~1h)
./zoho.sh resolve-account   # resolve accountId for ZOHO_EMAIL and save it
./zoho.sh folders           # list folders; copy the Inbox folderId into ZOHO_FOLDER_ID
```

Set the Inbox folder id (the `folderId=` for `/Inbox`):

```sh
sed -i 's/^ZOHO_FOLDER_ID=.*/ZOHO_FOLDER_ID=<INBOX_FOLDER_ID>/' zoho.env
```

---

## 6. Day-to-day use (refresh token already stored)

You normally do **not** need the browser again. Just refresh the access token and call the API:

```sh
./zoho.sh token            # refresh access token (~1h)
./zoho.sh list             # list Inbox (ZOHO_FOLDER_ID), newest first
./zoho.sh search "subject:verify" 10
./zoho.sh read <folderId> <messageId>
```

If the **refresh token is ever lost**, regenerate a grant code (section 5.2) and run
`./zoho.sh save-env && ./zoho.sh exchange`.

---

## 7. Agent recipe: wait for and read a verification email

End-to-end loop that triggers, polls, reads, and extracts a link/OTP:

```sh
set -e
source zoho.env

./zoho.sh token >/dev/null           # 1. fresh access token

# 2. trigger the action under test now (e.g. password reset to alpha1@tashira.tech)
TRIGGER_MS=$(date +%s%3N)

# 3. poll for the newest mail (~2 min, every 6s)
HIT=""
for i in $(seq 1 20); do
  OUT=$(./zoho.sh search "newMails" 20 2>/dev/null || true)
  HIT=$(printf '%s\n' "$OUT" | python3 - "$TRIGGER_MS" <<'PY'
import sys
trigger=int(sys.argv[1]); best=None
for line in sys.stdin:
    kw={}
    for kv in line.strip().split("\t"):
        if "=" in kv:
            k,v=kv.split("=",1); kw[k]=v
    try: rt=int(kw.get("receivedtime","0"))
    except: rt=0
    if rt>=trigger and (best is None or rt>best[0]):
        best=(rt, kw.get("folderId"), kw.get("messageId"), kw.get("subject"))
if best: print("%s\t%s\t%s" % (best[1], best[2], best[3]))
PY
)
  [ -n "$HIT" ] && break
  sleep 6
done

[ -z "$HIT" ] && { echo "no email found"; exit 1; }
FID=$(printf '%s' "$HIT" | cut -f1)
MID=$(printf '%s' "$HIT" | cut -f2)

# 4. read content and extract first link and a 4-8 digit OTP
./zoho.sh read "$FID" "$MID" | python3 <<'PY'
import sys,re,html
t=sys.stdin.read()
link=re.search(r'https?://[^\s"\'<>]+', t)
otp=re.search(r'\b(\d{4,8})\b', re.sub(r'<[^>]+>',' ',html.unescape(t)))
print("LINK:", link.group(0) if link else "-")
print("OTP :", otp.group(1) if otp else "-")
PY
```

Notes:
- `search "newMails"` returns only recent/unseen mail — ideal for "wait for newest".
- If you need a subject/other criteria, use Zoho search syntax, e.g. `search "subject:reset"`.
- To read **alpha2**, point `ZOHO_EMAIL`/tokens/`ZOHO_ACCOUNT_ID` at alpha2 (separate env set
  or run the bootstrap with alpha2's credentials) and repeat.

---

## 8. File: `.gitignore`

```
zoho.env
zoho.tokens.json
zoho.state.json
```

---

## 9. How the browser automation works (DOM ground truth)

Use this if you must adapt the bootstrap (e.g. different Zoho UI / data center).

Sign-in (two-step, avoids Apple SSO):
1. `goto https://api-console.zoho.com/` -> redirects to `accounts.zoho.com/signin`.
2. `#login_id` = email -> click `#nextbtn`.
3. `#password` = password -> click `#nextbtn`.
4. Wait until URL is `api-console.zoho.com` and not `signin`/`announcement`.
5. **Never** click "Sign in with Apple/OneAuth": a stray click makes smart sign-in default to
   Apple SSO (`appleid.apple.com`). The double `#nextbtn` flow avoids it.
6. Zoho enforces a **daily sign-in limit (~20/day)**; exceeding it redirects to
   `accounts.zoho.com/announcement/signin-block`. Avoid repeated logins -> reuse `--state`.

Self Client:
- Chooser card: `.clienttype_name` with text `Self Client`; its `CREATE NOW` opens `#selfclient`.
- Existing app appears under **Applications** as `Self Client <date>`; click it to open
  `https://api-console.zoho.com/client/<client_id>`.
- If none exists: click `CREATE` on the "Create New Client" form, confirm `OK`.

Credentials: click **Client Secret** tab; read inputs `#clientid` and `#clientsec`.

Grant code: on **Generate Code** tab fill `#scope_name` (comma-separated scopes), optional
`#scope_desc`, click `#generate_code`; the code is in input `#grant_token` (valid ~3 min).

---

## 10. Mail API reference (curl equivalents)

Auth header is `Authorization: Zoho-oauthtoken <token>` (NOT `Bearer`).

```sh
source zoho.env

# refresh access token
curl -s -X POST "https://${ZOHO_DC}/oauth/v2/token" \
  -d "grant_type=refresh_token" -d "client_id=${ZOHO_CLIENT_ID}" \
  -d "client_secret=${ZOHO_CLIENT_SECRET}" -d "refresh_token=${ZOHO_REFRESH_TOKEN}"

# accounts (-> accountId)
curl -s "https://${ZOHO_MAIL_BASE}/api/accounts" \
  -H "Accept: application/json" -H "Authorization: Zoho-oauthtoken ${ZOHO_ACCESS_TOKEN}"

# folders (-> folderId, e.g. Inbox)
curl -s "${ZOHO_MAIL_BASE}/api/accounts/${ZOHO_ACCOUNT_ID}/folders" \
  -H "Accept: application/json" -H "Authorization: Zoho-oauthtoken ${ZOHO_ACCESS_TOKEN}"

# list folder messages
curl -s "${ZOHO_MAIL_BASE}/api/accounts/${ZOHO_ACCOUNT_ID}/messages/view?folderId=${ZOHO_FOLDER_ID}&limit=50&sortorder=false" \
  -H "Accept: application/json" -H "Authorization: Zoho-oauthtoken ${ZOHO_ACCESS_TOKEN}"

# search
curl -s "${ZOHO_MAIL_BASE}/api/accounts/${ZOHO_ACCOUNT_ID}/messages/search?searchKey=newMails&limit=20&includeto=true" \
  -H "Accept: application/json" -H "Authorization: Zoho-oauthtoken ${ZOHO_ACCESS_TOKEN}"

# read content (HTML)
curl -s "${ZOHO_MAIL_BASE}/api/accounts/${ZOHO_ACCOUNT_ID}/folders/${ZOHO_FOLDER_ID}/messages/${MESSAGE_ID}/content?includeBlockContent=true" \
  -H "Accept: application/json" -H "Authorization: Zoho-oauthtoken ${ZOHO_ACCESS_TOKEN}"

# revoke refresh token (cleanup)
curl -s -X POST "https://${ZOHO_DC}/oauth/v2/token/revoke?token=${ZOHO_REFRESH_TOKEN}"
```

Key fields from list/search: `messageId`, `folderId`, `subject`, `fromAddress`, `summary`,
`receivedTime`, `hasAttachment`, `threadId`. The `content` endpoint returns HTML in `data.content`.
Access tokens live ~1h; refresh tokens do not expire (until revoked).

---

## 11. Troubleshooting

- **401 / empty `data`**: access token expired -> `./zoho.sh token`.
- **`invalid_code`**: grant code expired (~3 min) or reused -> regenerate (section 5.2).
- **`invalid_client`**: wrong Client ID/Secret or wrong DC -> check `zoho.env`, `ZOHO_DC`.
- **No `refresh_token`**: Server app needs `access_type=offline`; for Self Client the code may be
  consumed — regenerate.
- **Apple `appleid.apple.com` redirect during login**: smart sign-in chose Apple SSO (stray click).
  Use only the two-step `#login_id` -> `#nextbtn` -> `#password` -> `#nextbtn`; don't click SSO
  buttons; a fresh context (no stale `--state`) also helps.
- **`announcement/signin-block`**: daily sign-in limit hit (~20/day). Stop logging in; reuse
  `zoho.state.json`, or wait until the next day.
- **Region error / redirect**: org on another DC -> set `ZOHO_DC`/`ZOHO_MAIL_BASE` (section 12)
  and edit `BASE`/`ACCOUNTS` in `zoho_bootstrap.py`.
- **`scope_invalid`**: a scope isn't enabled for the plan/role -> drop it.

---

## 12. Appendix — data-center base URLs

| DC | Accounts server                | Mail host                  |
|----|--------------------------------|----------------------------|
| US | `https://accounts.zoho.com`    | `https://mail.zoho.com`    |
| EU | `https://accounts.zoho.eu`     | `https://mail.zoho.eu`     |
| IN | `https://accounts.zoho.in`     | `https://mail.zoho.in`     |
| AU | `https://accounts.zoho.com.au` | `https://mail.zoho.com.au` |
| JP | `https://accounts.zoho.jp`     | `https://mail.zoho.jp`     |
| UK | `https://accounts.zoho.uk`     | `https://mail.zoho.uk`     |
| CA | `https://accounts.zohocloud.ca`| `https://mail.zohocloud.ca`|
| SG | `https://accounts.zoho.sg`     | `https://mail.zoho.sg`     |
| AE | `https://accounts.zoho.ae`     | `https://mail.zoho.ae`     |
| SA | `https://accounts.zoho.sa`     | `https://mail.zoho.sa`     |

Set `ZOHO_DC` = accounts server URL and `ZOHO_MAIL_BASE` = mail host in `zoho.env`.
