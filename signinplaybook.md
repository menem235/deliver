# Standalone Runbook — Full login to https://agencies.semrush.com/

Everything needed to reproduce a successful, captcha-free login to the Semrush "Agency Partners platform" is contained in this single document: discovery evidence, all three scripts (copy-paste ready), exact run commands, expected outputs, and troubleshooting.

| Item | Value |
|---|---|
| Target | `https://agencies.semrush.com/` |
| Test account (researcher-owned) | `alpha1@tashira.tech` / `Donott@lktome2`  `alpha2@tashira.tech` / `Donott@lktome2`|
| Result | ✅ `POST https://www.semrush.com/sso/authorize` → **HTTP 201**, session verified |
| Captcha on login | **None** — no edge challenge ("Checking your browser - reCAPTCHA"), no form reCAPTCHA (`recaptcha_authorize: false` for this IP) |
| Runtime | Python 3 + Playwright (Chromium), headless |
| Artifacts dir | `./agencies-login-artifacts/` (created automatically by the scripts; screenshots + network JSON) |

---

## 0. Prerequisites

```bash
python3 -c "import playwright; print('playwright ok')"   # if this fails: pip install playwright && playwright install chromium
```
All commands below assume you are in an empty working directory. No other files are required — every script is defined in full, in this document.

---

## 1. How the login works (discovery evidence)

### 1.1 There is no `/login/` page

```bash
curl -sSI https://agencies.semrush.com/login   # 301 -> http://agencies.semrush.com/login/
curl -sS  https://agencies.semrush.com/login/  # 404 page titled "We got lost"
```

Every unknown path only receives a trailing-slash `301` (oddly to **http://**) and then the SPA's `404`. The site is a static Astro application whose scripts are hosted on `static.semrush.com` under `/agency-directory/listing/<content-hash>/_astro/…`.

### 1.2 Sign-in is a JS button that embeds the shared Semrush SSO

The homepage header contains these buttons:

```html
<button aria-label="Sign in" data-test-id="desktop-auth">…</button>
<button aria-label="Sign in" data-test-id="mobile-auth">…</button>
```

Clicking one dynamically imports `open-auth-popup.js` from `static.semrush.com`. The relevant source (captured during discovery):

```js
// https://static.semrush.com/agency-directory/listing/<hash>/_astro/open-auth-popup.<hash>.js
r = new i.default({
  popup: true,
  root: `${b(a)}/sso/`,                 // b = SSO root builder (see 1.3)
  defaultActiveForm: "register",        // modal opens on the REGISTER tab
  loginForm:  { showGoogleAuth: true, onSuccessCallback: … },
  regForm:    { showGoogleAuth: true, onSuccessCallback: …,
                data: { source: "agency_partners" } }
}, "production");
r.createForm("sso-form");               // renders the SSO modal in-page
```

Key facts baked into that config:
- The auth UI is the **same Semrush SSO component** used across semrush.com (email/password + "Continue with Google").
- `defaultActiveForm: "register"` ⇒ the modal opens on **Create your account**. You **must click the "Log in" link** ("Already have an account? Log in") before submitting credentials, otherwise you are submitting a signup (which is captcha-gated).
- `root` is the API base for every SSO call (see 1.3).

### 1.3 SSO root resolves to www.semrush.com

```js
// https://static.semrush.com/agency-directory/listing/<hash>/_astro/utils.<hash>.js
const t = "semrush.com";
// N(locale):  EN -> `https://www.${t}` ; other -> `https://${locale}.${t}`
export { N as b, … };
```

For the default (EN) locale: **`root = https://www.semrush.com`**, so the auth API calls are:

| Step | Endpoint | Behavior |
|---|---|---|
| flags | `POST https://www.semrush.com/sso/options?version=4.2.9` | returns `{"recaptcha_signup":true,"recaptcha_authorize":false,"recaptcha_restore":true}` |
| geo | `POST https://www.semrush.com/sso/country` | returns IP/consent info |
| **login** | **`POST https://www.semrush.com/sso/authorize`** | **201 = success**; `403 ERROR_LOGIN_WRONG_PASSWORD` = bad credentials; `400 ERROR_RECAPTCHA_NEED` = captcha required (not hit on login here) |
| register | `POST https://www.semrush.com/sso/signup?autologin=true` | requires captcha (`recaptcha_signup: true`) — **not used by this runbook** |

Observed response of the flags call during login:

```json
{"recaptcha_signup":true,"recaptcha_authorize":false,"recaptcha_restore":true}
```

### 1.4 Captcha / challenge behavior for this flow

- The modal always displays the reCAPTCHA **badge** (bottom-left) because the widget code is shared, but on the **Log in** tab the server flag `recaptcha_authorize` is `false` for this IP ⇒ no widget is executed, the POST to `/sso/authorize` carries **no** `g-recaptcha-response`, and the server accepts it (**201**).
- The **Finding-1 edge challenge** (page title `Checking your browser - reCAPTCHA`, sitekey `6Lfaz1gkAAAAAAeq7kxKii88L6SuaC3V3zjB_YQ7`) is monitored by the scripts at every stage (page load, modal open, login tab, post-submit polling) and **never appeared** on `agencies.semrush.com` or `www.semrush.com` during any run. If it ever appears, the scripts print `!!! EDGE CHALLENGE …` and save a screenshot so you can test the forged-cookie bypass (`Cookie: recaptcha-ca-e=1`) against it.
- The **register** tab would require a real captcha — irrelevant for logging in.

---

## 2. Script 1 (optional) — discovery

Saves to: **`agencies_login.py`**

```bash
cat > agencies_login.py <<'PYEOF'
import asyncio, json, os, traceback
from playwright.async_api import async_playwright
ROOT = os.environ.get("AG_ARTIFACTS", "./agencies-login-artifacts")
EMAIL = "alpha1@tashira.tech"; PASS = "Donott@lktome2"
async def main():
    net = []
    os.makedirs(ROOT, exist_ok=True)
    async with async_playwright() as p:
        ctx = await p.chromium.launch_persistent_context(
            user_data_dir=f"{ROOT}/profile_ag", headless=True,
            args=["--no-sandbox","--disable-dev-shm-usage"],
            user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36",
            viewport={"width":1440,"height":900})
        page = await ctx.new_page()
        async def on_resp(r):
            u = r.url
            if any(k in u for k in ("agencies.semrush.com","/sso/","auth","login","recaptcha")):
                try: body = (await r.text())[:200] if r.request.method != "GET" else ""
                except Exception: body = ""
                net.append({"m": r.request.method, "u": u[:180], "st": r.status, "body": body})
        page.on("response", lambda r: asyncio.ensure_future(on_resp(r)))
        try:
            await page.goto("https://agencies.semrush.com/", wait_until="domcontentloaded", timeout=45000)
            await page.wait_for_timeout(6000)
            await page.screenshot(path=f"{ROOT}/ag-home.png")
            print("home title:", await page.title())
            for sel in ['.ch2-deny-all-btn','.ch2-allow-all-btn','button:has-text("Deny")','button:has-text("Accept")']:
                try:
                    el = page.locator(sel)
                    if await el.count() and await el.first.is_visible(timeout=1000):
                        await el.first.click(timeout=3000); print("banner:", sel); break
                except Exception: pass
            await page.wait_for_timeout(800)
            clicked = False
            for sel in ['[data-test-id="desktop-auth"]','[data-test-id="mobile-auth"]','button[aria-label="Sign in"]']:
                try:
                    el = page.locator(sel).first
                    if await el.count():
                        await el.click(timeout=6000); print("clicked:", sel); clicked = True; break
                except Exception as e: print("click fail", sel, repr(e)[:100])
            if not clicked: print("NO SIGNIN BUTTON")
            await page.wait_for_timeout(5000)
            pages = ctx.pages
            print("open pages:", [pg.url for pg in pages])
            target = None
            for pg in pages:
                try:
                    if await pg.locator("input[type=email]").count():
                        target = pg; break
                except Exception: pass
            if not target and await page.locator("input[type=email]").count():
                target = page
            if not target:
                print("NO EMAIL INPUT FOUND")
            else:
                print("login form on:", target.url)
                await target.screenshot(path=f"{ROOT}/ag-loginform.png")
                t = await target.title()
                if "Checking your browser" in t: print("EDGE CHALLENGE BEFORE FILL")
                await target.locator("input[type=email]").first.fill(EMAIL)
                pw = target.locator("input[type=password]")
                if await pw.count(): await pw.first.fill(PASS)
                await target.screenshot(path=f"{ROOT}/ag-filled.png")
                for sel in ['button[type="submit"]','button:has-text("Sign in")','button:has-text("Log in")','button:has-text("Continue")','button:has-text("Next")']:
                    try:
                        el = target.locator(sel).first
                        if await el.count() and await el.is_visible():
                            await el.click(timeout=5000); print("submitted via", sel); break
                    except Exception as e: print("sub fail", repr(e)[:80])
                for i in range(12):
                    await target.wait_for_timeout(2500)
                    tt = await target.title()
                    body = (await target.inner_text("body"))[:250].replace("\n"," ")
                    print(f"t+{(i+1)*2.5}s title={tt[:50]!r} body={body[:150]!r}")
                    if "Checking your browser" in tt:
                        print("!!! EDGE CHALLENGE DURING LOGIN"); break
                    if any(k in body.lower() for k in ("incorrect","wrong","invalid")):
                        print("login error shown"); break
                await target.screenshot(path=f"{ROOT}/ag-final.png", full_page=True)
        except Exception:
            traceback.print_exc()
            try: await page.screenshot(path=f"{ROOT}/ag-crash.png")
            except Exception: pass
        print("NET:", json.dumps(net, indent=1)[:3000])
        open(f"{ROOT}/ag-net.json","w").write(json.dumps(net, indent=1))
        await ctx.close()
asyncio.run(main())
PYEOF
```

Run it:

```bash
timeout 180 python3 agencies_login.py
```

**What this run establishes (why script 2 exists):**
- Sign-in opens the SSO **modal in the same page** (plus a stray `about:blank` popup that can be ignored).
- The network capture proves the API root: `POST https://www.semrush.com/sso/options?version=4.2.9` → `{"recaptcha_signup":true,"recaptcha_authorize":false,"recaptcha_restore":true}`.
- The modal opens on the **register** tab, so a naive submit there attempts signup, not login. Script 2 switches to the **Log in** tab first.

---

## 3. Script 2 — the working login

Saves to: **`agencies_login2.py`**

Differences that make it succeed:
1. After the modal opens, clicks the **"Log in"** link to switch tabs.
2. Dumps form state (email/pwd present? recaptcha iframes? buttons?) before filling.
3. Checks for the edge challenge after every stage; aborts with a screenshot if found.
4. Polls the outcome for up to ~35 s and classifies error vs success.

```bash
cat > agencies_login2.py <<'PYEOF'
import asyncio, json, os, traceback
from playwright.async_api import async_playwright
ROOT = os.environ.get("AG_ARTIFACTS", "./agencies-login-artifacts")
EMAIL = "alpha1@tashira.tech"; PASS = "Donott@lktome2"
async def main():
    net = []
    os.makedirs(ROOT, exist_ok=True)
    async with async_playwright() as p:
        ctx = await p.chromium.launch_persistent_context(
            user_data_dir=f"{ROOT}/profile_ag2", headless=True,
            args=["--no-sandbox","--disable-dev-shm-usage"],
            user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36",
            viewport={"width":1440,"height":900})
        page = await ctx.new_page()
        async def on_resp(r):
            u = r.url
            if any(k in u for k in ("/sso/","agencies.semrush.com/api","authorize","recaptcha","Checking")):
                try: body = (await r.text())[:300] if (r.request.method == "POST" or "/authorize" in u or "/options" in u) else ""
                except Exception: body = ""
                net.append({"m": r.request.method, "u": u[:200], "st": r.status, "body": body})
        page.on("response", lambda r: asyncio.ensure_future(on_resp(r)))
        async def check_challenge(pg, tag):
            try:
                t = await pg.title()
                if "Checking your browser" in t:
                    print(f"!!! EDGE CHALLENGE [{tag}]: {pg.url}")
                    await pg.screenshot(path=f"{ROOT}/ag2-challenge-{tag}.png")
                    return True
            except Exception: pass
            return False
        try:
            await page.goto("https://agencies.semrush.com/", wait_until="domcontentloaded", timeout=45000)
            await page.wait_for_timeout(6000)
            for sel in ['.ch2-deny-all-btn','.ch2-allow-all-btn']:
                try:
                    el = page.locator(sel)
                    if await el.count() and await el.first.is_visible(timeout=1000):
                        await el.first.click(timeout=3000); break
                except Exception: pass
            await page.wait_for_timeout(500)
            # open SSO modal
            await page.locator('[data-test-id="desktop-auth"]').first.click(timeout=8000)
            await page.wait_for_timeout(5000)
            if await check_challenge(page, "modal-open"): raise SystemExit(0)
            # switch to Log in tab
            try:
                await page.get_by_text("Log in", exact=False).first.click(timeout=6000)
                print("clicked Log in link")
            except Exception as e: print("login link:", repr(e)[:120])
            await page.wait_for_timeout(3000)
            await page.screenshot(path=f"{ROOT}/ag2-logintab.png")
            # inspect form state
            state = await page.evaluate("""() => ({
                email: !!document.querySelector('input[type=email]'),
                pwd: !!document.querySelector('input[type=password]'),
                recaptchaIframes: [...document.querySelectorAll('iframe')].filter(f=>(f.src||'').includes('recaptcha')).map(f=>f.src.slice(0,130)),
                buttons: [...document.querySelectorAll('button')].map(b=>b.innerText.trim()).filter(t=>t&&t.length<30),
                body: document.body.innerText.replace(/\\s+/g,' ').slice(0,300)
            })""")
            print("STATE:", json.dumps(state, indent=1))
            if await check_challenge(page, "login-tab"): raise SystemExit(0)
            # fill credentials
            await page.locator("input[type=email]").first.fill(EMAIL)
            await page.locator("input[type=password]").first.fill(PASS)
            await page.wait_for_timeout(800)
            await page.screenshot(path=f"{ROOT}/ag2-filled.png")
            # submit
            sub = False
            for sel in ['button[type="submit"]','button:has-text("Log in")','button:has-text("Sign in")']:
                try:
                    el = page.locator(sel).first
                    if await el.count() and await el.is_visible():
                        await el.click(timeout=6000); print("submitted via", sel); sub = True; break
                except Exception as e: print("sub fail", repr(e)[:80])
            if not sub: print("NO SUBMIT BUTTON")
            # watch outcome
            for i in range(14):
                await page.wait_for_timeout(2500)
                if await check_challenge(page, f"after-submit-{i}"): break
                st = await page.evaluate("""() => ({
                    url: location.href,
                    title: document.title,
                    body: document.body.innerText.replace(/\\s+/g,' ').slice(0,260),
                    recaptcha: [...document.querySelectorAll('iframe')].filter(f=>(f.src||'').includes('recaptcha')).length
                })""")
                print(f"t+{(i+1)*2.5}s recaptcha_iframes={st['recaptcha']} url={st['url'][:90]} body={st['body'][:170]!r}")
                b = st['body'].lower()
                if any(k in b for k in ("wrong","incorrect","invalid","error","not found","disabled")):
                    print("LOGIN ERROR VISIBLE"); break
                if any(k in st['url'].lower() for k in ("dashboard","account","home")) or "log out" in b or "logout" in b:
                    print("LOGIN SUCCESS?"); break
            await page.screenshot(path=f"{ROOT}/ag2-final.png", full_page=True)
        except SystemExit:
            print("exited due to challenge")
        except Exception:
            traceback.print_exc()
            try: await page.screenshot(path=f"{ROOT}/ag2-crash.png")
            except Exception: pass
        interesting = [n for n in net if 'sso' in n['u'] or 'authorize' in n['u'] or '/api/' in n['u']]
        print("NET (sso/api):", json.dumps(interesting, indent=1)[:2500])
        open(f"{ROOT}/ag2-net.json","w").write(json.dumps(net, indent=1))
        await ctx.close()
asyncio.run(main())
PYEOF
```

Run it:

```bash
timeout 180 python3 agencies_login2.py
```

### Expected key output (actual result)

```
clicked Log in link
STATE: {
  "email": true,
  "pwd": true,
  "recaptchaIframes": [...],
  ...
}
submitted via button[type="submit"]
t+2.5s recaptcha_iframes=0 url=https://agencies.semrush.com/ body='Browse Agencies For Agencies FAQ ...'
...
NET (sso/api):
 {
  "m": "POST",
  "u": "https://www.semrush.com/sso/options?version=4.2.9",
  "st": 200,
  "body": "{\"recaptcha_signup\":true,\"recaptcha_authorize\":false,\"recaptcha_restore\":true}\n"
 },
 {
  "m": "POST",
  "u": "https://www.semrush.com/sso/authorize",
  "st": 201,          <-- LOGIN OK
  "body": ""
 }
```

Interpretation checklist:
- **`POST …/sso/authorize` → `201`** = credentials accepted, session cookie issued (HttpOnly, so invisible to `document.cookie`).
- **No `!!! EDGE CHALLENGE` lines** and **no `ag2-challenge-*.png`** files = the Finding-1 edge captcha never triggered.
- **`recaptcha_iframes=0` on the login tab** after submit = no form captcha was executed (`recaptcha_authorize:false`).
- The modal closes itself after success (`onSuccessCallback` → `window.location.reload()`), which is why the final screenshot shows the plain homepage again.

---

## 4. Script 3 — session verification

Reuses the **same persistent profile** (`profile_ag2`) created by script 2, so the session cookies are present, then calls the site's own authenticated API from inside the page.

Saves to: **`agencies_verify.py`**

```bash
cat > agencies_verify.py <<'PYEOF'
import asyncio, json, os
from playwright.async_api import async_playwright
ROOT = os.environ.get("AG_ARTIFACTS", "./agencies-login-artifacts")
async def main():
    os.makedirs(ROOT, exist_ok=True)
    async with async_playwright() as p:
        ctx = await p.chromium.launch_persistent_context(
            user_data_dir=f"{ROOT}/profile_ag2", headless=True,
            args=["--no-sandbox","--disable-dev-shm-usage"],
            user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36",
            viewport={"width":1440,"height":900})
        page = await ctx.new_page()
        await page.goto("https://agencies.semrush.com/", wait_until="domcontentloaded", timeout=45000)
        await page.wait_for_timeout(6000)
        r = await page.evaluate("""async () => {
            const out={};
            try{ const a=await fetch('/api/lead-profile/user/',{credentials:'include'}); out.leadProfile={st:a.status, body:(await a.text()).slice(0,300)}; }catch(e){ out.leadProfile=String(e); }
            try{ const b=await fetch('/api/account/feature/',{credentials:'include'}); out.feature={st:b.status, body:(await b.text()).slice(0,300)}; }catch(e){ out.feature=String(e); }
            out.cookieNames = document.cookie.split(';').map(c=>c.split('=')[0].trim());
            out.authText = document.body.innerText.replace(/\\s+/g,' ').slice(0,200);
            return out;
        }""")
        print(json.dumps(r, indent=1))
        for sel in ['[data-test-id="desktop-user"]','[aria-label="Log out"]','button:has-text("Log out")','[data-test-id="header-messages-icon"]']:
            try:
                c = await page.locator(sel).count()
                if c: print("found:", sel, c)
            except Exception: pass
        await page.screenshot(path=f"{ROOT}/ag2-verify.png")
        await ctx.close()
asyncio.run(main())
PYEOF
```

Run it:

```bash
timeout 90 python3 agencies_verify.py
```

### Expected output (actual result of the run)

```json
{
  "leadProfile": {
    "st": 200,
    "body": "{\"semrushId\":31610287,\"email\":\"alpha1@tashira.tech\",\"trafficType\":\"external\",\"subscriptionType\":null,\"userType\":\"free\",\"corporateType\":\"user\",\"tariff\":\"NONE\"}"
  },
  "feature": { "st": 200, "body": "[]" },
  "cookieNames": ["ref_code","refer_source","cookiehub","G_ENABLED_IDPS"]
}
```

The `semrushId` + account email inside `leadProfile` is the authoritative proof of an authenticated session. The real session cookies are `HttpOnly`, which is why they do not appear in `document.cookie`.

---

## 5. Complete reproduce sequence

```bash
# 1. save all three scripts (sections 2, 3, 4 above) into the current directory
# 2. run them in order:
timeout 180 python3 agencies_login.py     # optional discovery
timeout 180 python3 agencies_login2.py    # LOGIN  -> expect "st": 201 for /sso/authorize
timeout  90 python3 agencies_verify.py    # PROOF  -> expect semrushId + email JSON

# optional: override the artifacts directory
AG_ARTIFACTS=/tmp/ag-artifacts python3 agencies_login2.py
```

Start over with a clean session:

```bash
rm -rf ./agencies-login-artifacts
```

---

## 6. Artifacts produced (all inside `./agencies-login-artifacts/`)

Written automatically by the scripts — useful for review, not required to run anything:

| File | Content |
|---|---|
| `ag-home.png` | homepage before Sign in |
| `ag-loginform.png`, `ag-filled.png` | first attempt: modal on the **register** tab (captcha badge visible) |
| `ag2-logintab.png` | modal switched to the **Log in** tab |
| `ag2-filled.png` | credentials filled in the login form |
| `ag2-final.png` | post-login homepage (modal closed after the 201) |
| `ag2-verify.png` | verification-run screenshot |
| `ag2-net.json` | full captured network from the login run (includes `POST /sso/authorize → 201`) |
| `ag-net.json` | discovery-run network (`/sso/options`, `/sso/country`, agencies `/api/*` calls) |

---

## 7. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Cookie banner intercepts clicks | both scripts auto-dismiss it via `.ch2-deny-all-btn` / `.ch2-allow-all-btn` |
| Credentials submitted but nothing happens / a signup is attempted | you are on the **register** tab — click the `Log in` link first (script 2 does this automatically) |
| An `about:blank` popup opens and stays empty | harmless; the real form is the in-page modal (`createForm("sso-form")`) |
| `input[type=email]` fill times out | stale/corrupt persistent profile → `rm -rf ./agencies-login-artifacts/profile_ag2` and rerun script 2 |
| `POST /sso/authorize` returns `403` with `ERROR_LOGIN_WRONG_PASSWORD` | wrong credentials for this account |
| `POST /sso/authorize` returns `400` with `ERROR_RECAPTCHA_NEED` | the login flag flipped to `recaptcha_authorize:true` for your IP — the form would then need a solved reCAPTCHA token in `g-recaptcha-response` (not observed in these runs) |
| Want to test whether the Finding-1 edge captcha appears | scripts print `!!! EDGE CHALLENGE …` and save `ag2-challenge-*.png` if any page title is `Checking your browser - reCAPTCHA`; none occurred in any run — if it does appear, re-request with `Cookie: recaptcha-ca-e=1` (garbage value) to test the presence-only bypass |
| Registering instead of logging in | registration requires solving reCAPTCHA (`recaptcha_signup:true`); logging in does not (`recaptcha_authorize:false` from this IP) |
| `ModuleNotFoundError: playwright` | `pip install playwright && playwright install chromium` |
