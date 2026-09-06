#!/home/kali/fofa-venv/bin/python3.14
"""
fofa --create : create a FOFA/NOSEC account end-to-end and print the login.
Usage:
  fofa --create                 # full automated account creation
  fofa --create --brave-port N  # attach to Brave CDP on port N (default 9222)
  fofa --create --out FILE      # write creds JSON to FILE (default ~/fofa-accounts.json)

Requires:
  - A headed Brave running with --remote-debugging-port (launched separately, or
    via the PYTHON/brave launch line in the fofa-account-creation skill).
  - A playwright venv with a working greenlet (this env: /home/kali/fofa-venv/bin/python3.14).
  - Network access to i.nosec.org, tempinbox.xyz, endpoint.tempinbox.xyz.

Output: prints the FOFA credentials (email/password/username) to stdout and appends
them to the out file as JSON. Exit 0 on success (account confirmed + logged in).
"""
import sys, os, json, time, re, base64, random, string, argparse, subprocess, urllib.request, urllib.parse, threading

# Guards concurrent registry / archive writes when --create N runs in parallel.
_REGISTRY_LOCK = threading.Lock()

CDP_BASE = "http://127.0.0.1:9222"
REGISTER_URL = "https://i.nosec.org/register?locale=en&service=https://en.fofa.info/f_login?login_redirect_url=/"
LOGIN_URL = "https://i.nosec.org/login?service=https://en.fofa.info/f_login?login_redirect_url=/"
TEMPINBOX_API = "https://endpoint.tempinbox.xyz"
TEMPINBOX_WEB = "https://www.tempinbox.xyz/"


def http_get(url):
    req = urllib.request.Request(url, headers={"User-Agent": "Mozilla/5.0 Chrome/151", "Accept": "application/json,*/*"})
    with urllib.request.urlopen(req, timeout=20) as r:
        return r.status, r.read().decode(errors="replace")


def get_tempinbox_address():
    """Return a fresh xxx@tempinbox.xyz address on the FOFA-accepted tempinbox.xyz domain.
    IMPORTANT: only the 'tempinbox.xyz' domain is verified to pass FOFA's blocklist.
    /email/Random may return other tempinbox domains (e.g. thepiratebay.cloud) that FOFA rejects,
    so we pin the address to tempinbox.xyz explicitly via the /email/{email} endpoint."""
    u = "fofa" + "".join(random.choices(string.ascii_lowercase + string.digits, k=8))
    addr = u + "@tempinbox.xyz"
    url = TEMPINBOX_API + "/email/" + urllib.parse.quote(addr)
    try:
        st, body = http_get(url)
        if st == 200:
            try:
                # returns the json-encoded address string
                data = json.loads(body)
                if isinstance(data, str) and "@" in data:
                    return data
            except Exception:
                pass
    except Exception:
        pass
    return addr


def poll_tempinbox_browser(page, email, timeout_s=120):
    """Poll the tempinbox /mailbox session view (via the browser) for the FOFA confirmation token.

    IMPORTANT: tempinbox boxes created via the site (#create/#random) deliver via the /mailbox
    session view (session cookie 'email'), NOT via the endpoint.tempinbox.xyz API — the API returns
    [] for site-created boxes. So we read the DOM: open /mailbox, click the message, extract the
    confirmation_token from the raw page content. Returns (confirm_url, message_dict)."""
    import re as _re
    url = "https://www.tempinbox.xyz/mailbox"
    deadline = time.time() + timeout_s
    while time.time() < deadline:
        try:
            page.goto(url, wait_until="domcontentloaded", timeout=60000)
            time.sleep(4)
            # click the confirmation message if present
            try:
                page.click("text=Confirmation instructions")
            except Exception:
                pass
            time.sleep(3)
            html = page.content()
            tok_m = _re.search(r'confirmation_token=([A-Za-z0-9_\-]+)', html)
            if tok_m:
                confirm_url = "https://i.nosec.org/confirmation?confirmation_token=" + tok_m.group(1)
                # capture the full visible message (body + sender + date) for secure archival
                msg = {}
                try:
                    mbody = page.inner_text("body")
                    msg["sender"] = "no-reply@baimaohui.net"
                    msg["subject"] = "Confirmation instructions"
                    # grab date near the message
                    mdate = _re.search(r'\d{2} [A-Za-z]{3} \d{4} \d{2}:\d{2} [AP]M', mbody)
                    msg["date"] = mdate.group(0) if mdate else ""
                    msg["message"] = mbody[:2000]
                except Exception:
                    pass
                return confirm_url, msg
        except Exception as e:
            print("[!] mailbox poll retry:", str(e)[:50])
        time.sleep(6)
    return None, None


EASY_PASSWORD = "SecPass123!x"

# ============================ DORK HARVESTER ============================
# FOFA free tier: 300 web queries/mo, 3,000 web result-points/mo.
# The results UI shows 10 hosts per page; every page consumed costs 10 points
# => 3,000 points / 10 per page = 300 pages/month max across ALL dorks.
# (The REST API /api/v1/search/all requires a paid plan for free accounts:
#  free accounts get "[-700] Account Invalid", so harvesting runs through the
#  logged-in web session's result pages instead.)

FOFA_POINTS_FREE = 3000       # free monthly web result points
FOFA_POINTS_PER_PAGE = 10     # each results page (10 hosts) costs 10 points
QUOTA_FILE = os.path.expanduser("~/.fofa-accounts/fofa_quota.json")


def _quota_path(email=None):
    """Per-account quota file when an email is given (each account has its own
    3,000-point budget); falls back to the global file for shared-session runs."""
    if email:
        safe = re.sub(r'[^A-Za-z0-9._-]+', '_', email)
        return os.path.expanduser("~/.fofa-accounts/quota_%s.json" % safe)
    return QUOTA_FILE


def load_quota(email=None):
    path = _quota_path(email)
    try:
        with open(path) as f:
            d = json.load(f)
    except Exception:
        d = {}
    d.setdefault("period", time.strftime("%Y-%m"))
    d.setdefault("used_points", 0)
    if d.get("period") != time.strftime("%Y-%m"):
        d["period"] = time.strftime("%Y-%m")
        d["used_points"] = 0
    return d


def save_quota(d, email=None):
    path = _quota_path(email)
    os.makedirs(os.path.dirname(path), exist_ok=True)
    with open(path, "w") as f:
        json.dump(d, f, indent=2)


def quota_remaining(email=None):
    d = load_quota(email)
    return max(0, FOFA_POINTS_FREE - d["used_points"])


def _parse_count(s, default=0):
    try:
        return int(str(s).replace(",", ""))
    except Exception:
        return default


def _page_loaded(page):
    """A results page is loaded when result rows OR the 'N results' total text is present.
    (Service-style dorks like protocol=rdp render ip:port rows without <a> links; empty
    results render only the total.)"""
    try:
        if page.evaluate("""() => {
            for (const a of document.querySelectorAll('a[href]')) {
                const h = a.href;
                if (/^https?:\\/\\//.test(h) && !/fofa\\.info|x\\.com|twitter|t\\.me|github\\.com/i.test(h)) return true;
            }
            return false;
        }"""):
            return True
        body = page.inner_text("body")
        return bool(re.search(r'([\d,]+)\s+results', body))
    except Exception:
        return False


def _page_total(page):
    try:
        body = page.inner_text("body")
        m = re.search(r'([\d,]+)\s+results', body)
        if m:
            return int(m.group(1).replace(",", ""))
    except Exception:
        pass
    return None


def harvest_dork_download(page, dork, want=100, out_dir=None, dl_dir=None, email=None):
    """Harvest hosts for one dork using FOFA's official Download Results function.

    Free tier: 3,000 result-points/month. The download dialog consumes
    '1 result = 1 F point' from the FREE CREDIT; a download only succeeds when
    requested count <= remaining free credit (else '[820031] F Points Insufficient').

    Flow: open results page -> open Download dialog -> read live free credit ->
    set count -> submit -> poll /userInfo/downloadRecords until the new export is
    'Available' -> download its CSV -> parse hosts.

    want: how many results to request for this dork (capped by free credit).
    email: attribute the quota usage to this account (per-account budget).
    Returns (hosts_list, downloaded_count, total_reported)."""
    qb = base64.b64encode(dork.encode()).decode()
    out_dir = out_dir or os.path.expanduser("~/.fofa-accounts/dorks")
    os.makedirs(out_dir, exist_ok=True)

    # ---- open the results page for the dork ----
    url = f"https://en.fofa.info/result?qbase64={qb}"
    last_err = None
    loaded = False
    for attempt in range(3):
        try:
            page.goto(url, wait_until="domcontentloaded", timeout=60000)
            time.sleep(3)
            for _ in range(15):
                if _page_loaded(page):
                    loaded = True
                    break
                time.sleep(1)
            if loaded:
                break
        except Exception as e:
            last_err = e
            time.sleep(3)
    if not loaded:
        raise RuntimeError("results page failed to load: %s" % last_err)

    total = _page_total(page)
    if total == 0:
        print("    [i] 0 results for this dork — nothing to download")
        with open(os.path.join(out_dir, re.sub(r'[^A-Za-z0-9._-]+', '_', dork)[:80] + ".txt"), "w") as f:
            pass
        return [], 0, 0

    # ---- snapshot existing download records so we can identify the NEW one ----
    def list_records():
        page.goto("https://en.fofa.info/userInfo/downloadRecords",
                  wait_until="domcontentloaded", timeout=45000)
        time.sleep(4)
        return page.evaluate("""() => {
            const rows = [...document.querySelectorAll('a')].filter(a => /Click to Download/.test(a.innerText||''));
            return rows.map(a => ({href: a.href}));
        }""")

    before = {r["href"] for r in list_records()}

    # ---- open the results page again + the Download dialog ----
    page.goto(url, wait_until="domcontentloaded", timeout=60000)
    time.sleep(4)
    page.keyboard.press("End")
    time.sleep(2)
    page.evaluate("() => document.querySelector('.icon-down-011')?.closest('button')?.click()")
    time.sleep(3)
    dlg = page.evaluate("""() => {
        const d = [...document.querySelectorAll('[aria-label="Download Results"], .el-dialog, .el-overlay-dialog')]
            .find(x => x.offsetWidth || x.offsetHeight);
        if (!d) return null;
        const m = (d.innerText || '').match(/Free Credit:\\s*([\\d,]+)/);
        return {free_credit: m ? m[1].replace(/,/g, '') : null};
    }""")
    if not dlg:
        raise RuntimeError("Download Results dialog did not open")
    free_credit = _parse_count(dlg.get("free_credit"), 0)

    remaining = quota_remaining(email)
    # The dialog credit is the source of truth. If the local tracker drifted (manual downloads
    # elsewhere, earlier debug runs), resync it DOWN to FOFA's number.
    if free_credit < remaining:
        drift = remaining - free_credit
        quota = load_quota(email)
        quota["used_points"] = min(FOFA_POINTS_FREE, quota["used_points"] + drift)
        save_quota(quota, email)
        print("    [i] resynced quota tracker to FOFA's free credit (%d left; drift %d)"
              % (free_credit, drift))
        remaining = free_credit
    n = min(want if want > 0 else 10**9, free_credit, remaining if remaining > 0 else 0,
            total if total else want)
    if n <= 0:
        print("    [!] no credit left: dialog free-credit=%s tracked-remaining=%d" % (free_credit, remaining))
        return [], 0, total
    if n < want:
        print("    [i] capping request to %d (want %d; credit %d, tracked %d, total %s)"
              % (n, want, free_credit, remaining, total))

    # ---- set the count and submit ----
    page.evaluate("""(n) => {
        const d = [...document.querySelectorAll('[aria-label="Download Results"], .el-dialog, .el-overlay-dialog')]
            .find(x => x.offsetWidth || x.offsetHeight);
        const inp = d.querySelector('input[type=number]');
        const setter = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set;
        setter.call(inp, String(n));
        inp.dispatchEvent(new Event('input', {bubbles: true}));
        inp.dispatchEvent(new Event('change', {bubbles: true}));
    }""", n)
    time.sleep(1)
    page.evaluate("""() => {
        const d = [...document.querySelectorAll('[aria-label="Download Results"], .el-dialog, .el-overlay-dialog')]
            .find(x => x.offsetWidth || x.offsetHeight);
        const b = [...d.querySelectorAll('button')].find(b => (b.innerText || '').trim() === 'Download');
        b.click();
    }""")
    time.sleep(4)

    # ---- poll the records page until a NEW record appears (export is async) ----
    new_rec = None
    for _ in range(20):  # up to ~2.5 min
        recs = list_records()
        fresh = [r for r in recs if r["href"] not in before]
        if fresh:
            new_rec = fresh[0]
            break
        time.sleep(7)
    if not new_rec:
        raise RuntimeError("download request did not produce a record (check Download History)")

    # ---- fetch the CSV via the session ----
    fname = new_rec["href"].rstrip("/").split("/")[-1] or ("fofa_%s.csv" % re.sub(r'[^A-Za-z0-9._-]+', '_', dork)[:40])
    dl_dir = dl_dir or out_dir
    os.makedirs(dl_dir, exist_ok=True)
    raw_path = os.path.join(dl_dir, fname)
    with page.expect_download(timeout=120000) as dl_info:
        page.evaluate("(href) => { const a=[...document.querySelectorAll('a')].find(a => a.href === href); a.click(); }",
                      new_rec["href"])
    dl_info.value.save_as(raw_path)

    # ---- parse: first column is host ----
    hosts = []
    with open(raw_path, encoding="utf-8-sig", errors="replace") as f:
        lines = [l for l in f.read().splitlines() if l.strip()]
    for line in (lines[1:] if len(lines) > 1 else lines):
        parts = line.split(",")
        if parts:
            hosts.append(parts[0].strip().strip('"'))
    hosts = [h for h in hosts if h]

    # ---- account the quota (1 result = 1 point) ----
    quota = load_quota(email)
    quota["used_points"] += len(hosts) if hosts else n
    save_quota(quota, email)

    # ---- save per-dork output (plain host list) ----
    safe = re.sub(r'[^A-Za-z0-9._-]+', '_', dork)[:80]
    out_file = os.path.join(out_dir, safe + ".txt")
    with open(out_file, "w") as f:
        f.write("\n".join(hosts) + ("\n" if hosts else ""))
    print("    [+] %d hosts downloaded -> %s (total reported: %s; csv: %s)"
          % (len(hosts), out_file, total, fname))
    return hosts, len(hosts), total


def dorks_from_session(session_json=None):
    """Return (email, key, session) for the most recent account with session info."""
    recs = []
    session_json = session_json or os.path.expanduser("~/fofa-accounts.json")
    if os.path.exists(session_json):
        try:
            recs = json.load(open(session_json))
        except Exception:
            recs = []
    recs = [r for r in recs if r.get("session") and r["session"].get("brave_port")]
    if not recs:
        return None, None, None
    return recs[-1].get("email"), recs[-1].get("password"), recs[-1]["session"]


def run_dorks(dorks, brave_port, want=100, out_dir=None, quota_cap_points=None):
    """Run a list of dorks through one logged-in session using the Download function.
    want = results to download per dork. Returns dict dork->hosts."""
    from playwright.sync_api import sync_playwright
    results = {}
    with sync_playwright() as pw:
        browser = pw.chromium.connect_over_cdp("http://127.0.0.1:%d" % brave_port)
        ctx = browser.contexts[0]
        page = ctx.pages[0] if ctx.pages else ctx.new_page()
        for i, dork in enumerate(dorks, 1):
            print("[*] dork %d/%d: %s (want %s)" % (i, len(dorks), dork, want if want else "max"))
            if quota_cap_points is not None and quota_remaining() <= quota_cap_points:
                print("[!] stopping: quota cap reached")
                break
            try:
                hosts, got, total = harvest_dork_download(page, dork, want=want, out_dir=out_dir)
                results[dork] = hosts
            except Exception as e:
                print("    [!] dork failed: %s" % str(e)[:80])
                results[dork] = []
        browser.close()
    return results


def run_dorks_max_per_account(dorks, password=EASY_PASSWORD, out_dir=None, parallel=2,
                              registry=None, keep_braves=True):
    """One FRESH account per dork; each account downloads as many hosts as its free
    credit allows (want=0 -> capped only by credit/result-total). Accounts are minted
    `parallel` at a time, each in its own isolated Brave on its own CDP port.

    Returns list of {dork, email, brave_port, profile_dir, hosts, total}."""
    from playwright.sync_api import sync_playwright
    out_dir = out_dir or os.path.expanduser("~/.fofa-accounts/dorks")
    registry = registry or os.path.expanduser("~/fofa-accounts.json")
    results = []

    def one(idx_dork):
        idx, dork = idx_dork
        tag = "dork %d/%d [%s]" % (idx + 1, len(dorks), dork[:40])
        # ---- mint a fresh account on its own session ----
        port = find_free_port()
        if port is None:
            print("[%s] no free port" % tag)
            return {"dork": dork, "email": None, "hosts": [], "total": None, "error": "no port"}
        proc, profile_dir = launch_fresh_brave(port)
        print("[%s] fresh account/browser on port %d" % (tag, port))
        email = None
        hosts, total = [], None
        try:
            ok, creds = run_create(port, registry, password)
            if not ok or not creds:
                raise RuntimeError("account creation failed")
            email = creds["email"]

            # ---- max download for this dork on this account ----
            with sync_playwright() as pw:
                browser = pw.chromium.connect_over_cdp("http://127.0.0.1:%d" % port)
                ctx = browser.contexts[0]
                page = ctx.pages[0] if ctx.pages else ctx.new_page()
                hosts, got, total = harvest_dork_download(page, dork, want=0, out_dir=out_dir,
                                                          email=email)
                browser.close()
            print("[%s] DONE: %d hosts (total on FOFA: %s)" % (tag, len(hosts), total))
            return {"dork": dork, "email": email, "brave_port": port,
                    "profile_dir": profile_dir, "hosts": hosts, "total": total}
        except Exception as e:
            print("[%s] FAILED: %s" % (tag, str(e)[:90]))
            return {"dork": dork, "email": email, "brave_port": port,
                    "profile_dir": profile_dir, "hosts": [], "total": total,
                    "error": str(e)[:120]}
        finally:
            if not keep_braves and proc:
                try:
                    proc.terminate()
                except Exception:
                    pass

    import concurrent.futures as _cf
    with _cf.ThreadPoolExecutor(max_workers=max(1, parallel)) as ex:
        for res in ex.map(one, list(enumerate(dorks))):
            results.append(res)

    ok = [r for r in results if r.get("hosts")]
    print("\n[OK] %d/%d dorks harvested, %d hosts total" % (len(ok), len(dorks),
                                                            sum(len(r["hosts"]) for r in results)))
    for r in results:
        print("    %-45s -> %5d hosts (total %s) via %s" % (
            r["dork"][:45], len(r["hosts"]), r.get("total"), r.get("email")))
    return results


# ========================== /DORK HARVESTER ===========================


def run_create(brave_port, out_file, password=EASY_PASSWORD, venv_python=None):
    from playwright.sync_api import sync_playwright

    cdp = "http://127.0.0.1:%d" % brave_port
    username = "secpoc" + "".join(random.choices(string.ascii_lowercase, k=6))

    with sync_playwright() as pw:
        browser = pw.chromium.connect_over_cdp(cdp)
        ctx = browser.contexts[0]
        page = ctx.pages[0] if ctx.pages else ctx.new_page()

        # Generate a real tempinbox address on the FOFA-accepted tempinbox.xyz domain.
        # The site only delivers mail to addresses it generated (via #create/#random with #domain).
        # Kept #domain = tempinbox.xyz explicitly and use #create so the result is @tempinbox.xyz
        # (Random rotates to other tempinbox domains that FOFA rejects).
        import re as _re
        page.goto(TEMPINBOX_WEB, wait_until="domcontentloaded", timeout=60000)
        time.sleep(6)
        user = "fofagen" + "".join(random.choices(string.ascii_lowercase, k=6))
        # Set user + domain, then click Create. Do NOT let a prefill timeout abort the create.
        try:
            page.fill("#user", user)
        except Exception as e:
            print("[!] tempinbox user fill:", str(e)[:50])
        try:
            d = page.query_selector("#domain")
            if d:
                d.evaluate("el => { el.value = 'tempinbox.xyz'; el.dispatchEvent(new Event('input',{bubbles:true})); el.dispatchEvent(new Event('change',{bubbles:true})); }")
        except Exception as e:
            print("[!] tempinbox domain set:", str(e)[:50])
        time.sleep(1)
        try:
            page.click("#create")
        except Exception as e:
            print("[!] tempinbox create:", str(e)[:50])
        time.sleep(7)
        # The create used our user + forced domain tempinbox.xyz, so the address is predictable:
        email = user + "@tempinbox.xyz"
        # Confirm it's actually registered by checking the /mailbox view lists it as active.
        try:
            page.goto("https://www.tempinbox.xyz/mailbox", wait_until="domcontentloaded", timeout=60000)
            time.sleep(4)
            mbody = page.inner_text("body")
            if email not in mbody:
                # fall back to whichever @tempinbox.xyz the mailbox shows (active box)
                m = _re.findall(r'[\w.+-]+@tempinbox\.xyz', mbody)
                if m:
                    email = m[0]
        except Exception as e:
            print("[!] tempinbox mailbox verify:", str(e)[:50])
        print("[*] email:", email)
        print("[*] username:", username)

        def go(url, tries=3):
            for i in range(tries):
                try:
                    page.goto(url, wait_until="domcontentloaded", timeout=60000)
                    time.sleep(3)
                    return True
                except Exception as e:
                    print("[!] nav retry %d on %s (%s)" % (i + 1, url.split("/")[2][:30], str(e)[:50]))
                    time.sleep(3)
            return False

        # --- REGISTER ---
        go(REGISTER_URL)
        time.sleep(4)
        # Solve Cloudflare Turnstile (auto-solves but intermittently "Failed to verify";
        # solve_turnstile reloads and retries until the token populates, then we submit only when solved).
        ts_ok, ts_token = solve_turnstile(page, REGISTER_URL)
        if not ts_ok:
            print("[!] could not solve Turnstile after retries; aborting")
            browser.close()
            return False, None
        # clear honeypot, check agree (guard for None on nav failure)
        honey = page.query_selector("#nosectest")
        if honey:
            honey.evaluate("e => e.value = ''")
        agree = page.query_selector("#agree")
        if agree:
            agree.evaluate("e => { if(!e.checked) e.click(); }")
        # if the register page didn't load (DNS fail / interstitial), bail cleanly
        if not page.query_selector("#nosecuser_email"):
            print("[!] register form not present (page may have failed to load)")
            browser.close()
            return False, None
        page.fill("#nosecuser_email", email)
        page.fill("#nosecuser_password", password)
        page.fill("#nosecuser_password_confirmation", password)
        page.fill("#nosecuser_username", username)
        try:
            with page.expect_navigation(timeout=30000):
                page.click("input[type=submit]")
        except Exception:
            pass
        time.sleep(5)
        body = page.inner_text("body")
        if "confirmation link has been sent" in body:
            print("[+] register accepted — activation email sent")
        elif "not supported" in body:
            print("[!] FOFA rejected the email domain:", email.split("@")[1], "— try again (tempinbox should pass)")
            browser.close()
            return False, None
        else:
            print("[?] register response:", body[:120].replace("\n", " / ")[:120])

        # --- POLL TEMPINBOX (via browser /mailbox DOM) ---
        print("[*] polling tempinbox /mailbox for activation email...")
        confirm_url, activation_msg = poll_tempinbox_browser(page, email)
        if not confirm_url:
            print("[!] activation email not found in tempinbox within 120s")
            browser.close()
            return False, None
        print("[+] got confirmation link:", confirm_url)

        # --- ACTIVATE ---
        go(confirm_url)
        time.sleep(5)
        body = page.inner_text("body")
        if "successfully confirmed" in body:
            print("[+] account CONFIRMED")
        else:
            print("[?] confirm response:", body[:120].replace("\n", " / ")[:120])

        # --- LOGIN ---
        logged_in, final_url, title = do_login(page, email, password, LOGIN_URL)
        print("[+] login ->", final_url, "| title:", title, "| logged_in:", logged_in)

        creds = {"email": email, "password": password, "username": username,
                 "confirm_url": confirm_url, "final_url": final_url,
                 "logged_in": logged_in, "created": time.strftime("%Y-%m-%d %H:%M:%S"),
                 "activation_email": activation_msg}
        browser.close()

        # append to out file (registry) — thread-safe for concurrent --create N
        with _REGISTRY_LOCK:
            records = []
            if os.path.exists(out_file):
                try:
                    with open(out_file) as f:
                        records = json.load(f)
                except Exception:
                    records = []
            records.append(creds)
            with open(out_file, "w") as f:
                json.dump(records, f, indent=2)
        print("[+] saved registry ->", out_file)

        return logged_in, creds


def turnstile_token_len(page):
    """Return the length of the populated cf-turnstile-response value, or -1 if the input is absent."""
    toks = page.query_selector("[name=cf-turnstile-response]")
    if not toks:
        return -1
    v = toks.evaluate("e => e.value")
    return len(v) if v else 0


def solve_turnstile(page, register_url, max_reloads=4, wait_s=15):
    """Ensure Cloudflare Turnstile is solved (token populated) before posting the form.

    The widget intermittently shows 'Failed to verify' and never fills the token. A page reload
    re-runs the widget and usually solves it. Loop: reload, wait up to wait_s for the token, repeat
    up to max_reloads. Returns (solved_bool, token_str)."""
    for attempt in range(max_reloads):
        try:
            page.goto(register_url, wait_until="domcontentloaded", timeout=60000)
        except Exception as e:
            print("[!] turnstile nav retry:", str(e)[:50])
        time.sleep(3)
        for _ in range(wait_s):
            ln = turnstile_token_len(page)
            if ln > 20:
                tok = page.query_selector("[name=cf-turnstile-response]")
                return True, (tok.evaluate("e => e.value") if tok else "").strip()
            time.sleep(1)
        print("[!] turnstile not solved (attempt %d/%d) -> reloading" % (attempt + 1, max_reloads))
    return False, ""


def do_login(page, email, password, login_url, max_attempts=4):
    """Log into FOFA. The login form has its own Turnstile which also intermittently fails, so we
    reload + re-fill + re-solve up to max_attempts, clicking the real button[type=submit] (text Login).
    Returns (logged_in_bool, final_url, title)."""
    logged_in = False
    final_url = ""
    title = ""
    for attempt in range(max_attempts):
        try:
            page.goto(login_url, wait_until="domcontentloaded", timeout=60000)
        except Exception as e:
            print("[!] login nav retry:", str(e)[:50])
        time.sleep(4)
        # if we're already on the FOFA dashboard (profile logged in), treat as success and stop
        title = page.title()
        final_url = page.url
        body = page.inner_text("body")
        if ("fofa.info" in final_url and "login" not in final_url.lower()) or "FOFA Search Engine" in body:
            logged_in = True
            return True, final_url, title
        # fill creds
        if not page.query_selector("#username"):
            print("[!] login form not present; reloading")
            continue
        page.fill("#username", email)
        page.fill("#password", password)
        try:
            page.query_selector("#fofa_service").evaluate("e => { if(!e.checked) e.click(); }")
        except Exception:
            pass
        # solve turnstile (reload+wait if needed)
        ts_ok = False
        for _ in range(15):
            ln = turnstile_token_len(page)
            if ln > 20:
                ts_ok = True
                break
            time.sleep(1)
        if not ts_ok:
            print("[!] login turnstile not solved (attempt %d/%d) -> reload to retry" % (attempt + 1, max_attempts))
            continue
        # click the real submit button via JS
        try:
            with page.expect_navigation(timeout=30000):
                page.evaluate("""() => {
                    const f = document.querySelector('form');
                    const b = f ? (f.querySelector('button[type=submit]') || f.querySelector('input[type=submit]')) : null;
                    if (b) { b.click(); return 'clicked'; }
                    if (f) { f.submit(); return 'form.submit'; }
                    return 'no form';
                }""")
        except Exception:
            pass
        time.sleep(6)
        # wait for the redirect / dashboard
        for i in range(8):
            final_url = page.url
            title = page.title()
            body = page.inner_text("body")
            if ("fofa.info" in final_url and "login" not in final_url.lower()) or \
               "FOFA Search Engine" in body or "Logging in" in body or "Asset Reward" in body:
                logged_in = True
                break
            time.sleep(3)
        if logged_in:
            return True, final_url, title
    return logged_in, final_url, title


_PORT_COUNTER = [9499]  # monotonic allocator; each call hands out the NEXT-lower port (unique)


def find_free_port(start=9400, end=9499):
    """Return a free TCP port in [start,end]. UNIQUE per call (monotonic counter), so concurrent
    threads can never select the same port even if Brave hasn't finished binding yet. The counter
    walks DOWNWARD from a high base; if a port is already taken (leftover Brave), the caller's own
    Brave just fails to bind and that account retries — callers always get distinct port numbers."""
    with _REGISTRY_LOCK:
        p = _PORT_COUNTER[0]
        _PORT_COUNTER[0] = p - 1
        if p < start:
            return None
        return p


def launch_fresh_brave(port):
    """Spawn a fresh Brave on the given CDP port with a clean profile dir; return (pid, profile_dir).
    A fresh profile guarantees no prior logged-in FOFA session, so /login shows the real form."""
    import tempfile
    profile_dir = tempfile.mkdtemp(prefix="fofa-brave-")
    cmd = ["/usr/bin/brave-browser",
           "--user-data-dir=" + profile_dir,
           "--remote-debugging-port=%d" % port,
           "--no-first-run", "--no-default-browser-check", "--disable-brave-update",
           "--window-size=1400,900", "about:blank"]
    with open(os.devnull, "w") as dn:
        proc = subprocess.Popen(cmd, stdout=dn, stderr=dn, env={**os.environ, "DISPLAY": os.environ.get("DISPLAY", ":0")})
    # wait for CDP to come up
    for _ in range(30):
        try:
            import urllib.request
            with urllib.request.urlopen("http://127.0.0.1:%d/json/version" % port, timeout=2) as r:
                if r.status == 200:
                    return proc, profile_dir
        except Exception:
            pass
        time.sleep(1)
    return proc, profile_dir


def main():
    ap = argparse.ArgumentParser(prog="fofa", description="FOFA account automation")
    ap.add_argument("--create", default="1", help="create N FOFA accounts (default 1). Each account gets its OWN isolated browser session.")
    ap.add_argument("--brave-port", default=None, help="attach to ONE existing Brave CDP port. When creating >1 account, prefer no --brave-port so each gets its own.")
    ap.add_argument("--out", default=os.path.expanduser("~/fofa-accounts.json"), help="creds output JSON")
    ap.add_argument("--python", default=None, help="python interpreter with playwright (auto-detect if omitted)")
    ap.add_argument("--password", default=EASY_PASSWORD, help="FOFA account password (default: the shared easy password)")
    ap.add_argument("--secure-dir", default=None, help="dir to save each account's creds+email (default ~/.fofa-accounts, env FOFA_SECURE_DIR)")
    ap.add_argument("--keep-sessions", action="store_true", help="leave each account's Brave OPEN after creation and record its CDP port+profile dir in the archive")
    ap.add_argument("--keep-brave", action="store_true", help="alias for --keep-sessions (keeps the auto-launched Braven running)")
    # dork harvester
    ap.add_argument("--dork", default=None, metavar='"QUERY"', help='search FOFA for a dork (e.g. --dork \'title="crawl4ai"\') and download the hosts')
    ap.add_argument("--dorks-file", default=None, metavar="FILE", help="file with one dork per line (combined with/instead of --dork)")
    ap.add_argument("--dork-port", type=int, default=None, help="CDP port of the logged-in Brave to harvest with (default: newest kept session)")
    ap.add_argument("--dork-count", type=int, default=100, help="results to download per dork via the Download function (default 100; capped by free credit)")
    ap.add_argument("--dork-pages", type=int, default=None, help="(legacy) pages per dork; converted to --dork-count = pages*10")
    ap.add_argument("--dork-out", default=None, metavar="DIR", help="dir for per-dork host files (default ~/.fofa-accounts/dorks)")
    ap.add_argument("--dork-cap", type=int, default=None, metavar="POINTS", help="stop when remaining quota points reach this floor")
    ap.add_argument("--dorks-max", action="store_true", help="one FRESH account per dork; each downloads the MAXIMUM its free credit allows (~2900+ hosts)")
    ap.add_argument("--parallel", type=int, default=2, metavar="N", help="with --dorks-max: accounts minted concurrently (default 2)")
    ap.add_argument("--quota", action="store_true", help="print the tracked monthly quota and exit")
    args = ap.parse_args()

    if args.secure_dir:
        os.environ["FOFA_SECURE_DIR"] = args.secure_dir

    # ---- dork-harvester mode ----
    if args.dork or args.dorks_file or args.quota:
        if args.quota:
            d = load_quota()
            print("quota period:      %s" % d["period"])
            print("points used:       %d / %d" % (d["used_points"], FOFA_POINTS_FREE))
            print("points remaining:  %d  (= %d pages of 10 hosts)" % (quota_remaining(), quota_remaining() // FOFA_POINTS_PER_PAGE))
            sys.exit(0)

        dorks = []
        if args.dork:
            dorks.append(args.dork)
        if args.dorks_file:
            with open(args.dorks_file) as f:
                dorks += [l.strip() for l in f if l.strip() and not l.strip().startswith("#")]
        if not dorks:
            print("[!] no dorks given (use --dork or --dorks-file)")
            sys.exit(1)

        if args.dorks_max:
            # ONE FRESH ACCOUNT PER DORK, each draining its full free credit (~2900+ hosts)
            res = run_dorks_max_per_account(dorks, password=args.password, out_dir=args.dork_out,
                                            parallel=args.parallel)
            out_json = os.path.join(args.dork_out or os.path.expanduser("~/.fofa-accounts/dorks"),
                                    "_dorks_max_summary.json")
            try:
                os.makedirs(os.path.dirname(out_json), exist_ok=True)
                with open(out_json, "w") as f:
                    json.dump(res, f, indent=2)
                print("[+] summary -> %s" % out_json)
            except Exception as e:
                print("[!] summary save failed: %s" % str(e)[:60])
            sys.exit(0 if any(r.get("hosts") for r in res) else 1)

        port = args.dork_port
        if port is None:
            _, _, sess = dorks_from_session()
            if not sess:
                print("[!] no kept session found. Run: fofa --create 1 --keep-sessions  (then retry the dork)")
                sys.exit(1)
            port = sess["brave_port"]
            print("[*] using kept session on CDP port %d (profile %s)" % (port, sess.get("profile_dir")))
        print("[*] quota: %d points remaining (%d pages)" % (quota_remaining(), quota_remaining() // FOFA_POINTS_PER_PAGE))
        want = args.dork_count
        if args.dork_pages is not None:
            want = args.dork_pages * FOFA_POINTS_PER_PAGE  # legacy: pages -> results
        results = run_dorks(dorks, port, want=want, out_dir=args.dork_out,
                            quota_cap_points=(args.dork_cap if args.dork_cap is not None else 0))
        total_hosts = sum(len(v) for v in results.values())
        print("\n[OK] %d dork(s), %d hosts total. Files in %s" % (
            len(results), total_hosts, args.dork_out or os.path.expanduser("~/.fofa-accounts/dorks")))
        sys.exit(0)

    # parse the count: --create 5   (also accept --create 5 or bare --create -> 1)
    try:
        count = int(args.create)
    except ValueError:
        count = 1
    count = max(1, count)

    py = args.python
    if not py:
        for cand in ["/home/kali/fofa-venv/bin/python3.14", "/home/kali/pb/bin/python3.13"]:
            if os.path.exists(cand):
                py = cand
                break

    # If --brave-port given, that one Brave is shared across ALL accounts (warning for >1).
    shared_port = int(args.brave_port) if args.brave_port else None
    if shared_port and count > 1:
        print("[!] --brave-port shares ONE session across %d accounts; each account will not be isolated." % count)
        print("[!] omit --brave-port to give each account its own isolated browser session.")

    keep = args.keep_sessions or args.keep_brave

    import concurrent.futures as _cf

    def create_one(idx):
        """Create ONE account in its OWN isolated Brave session. Run concurrently per account."""
        print("=" * 60)
        print("[*] creating account %d/%d" % (idx + 1, count))
        print("=" * 60)
        owned_proc = None
        profile_dir = None
        if shared_port:
            brave_port = shared_port
        else:
            port = find_free_port()
            if port is None:
                print("[!] account %d: no free port found" % (idx + 1))
                return False, None
            owned_proc, profile_dir = launch_fresh_brave(port)
            brave_port = port
            print("[*] account %d fresh Brave on CDP port %d (profile: %s)" % (idx + 1, port, profile_dir))

        ok, creds = run_create(brave_port, args.out, args.password, py)

        # record session info so each account's browser can be reused independently
        if creds:
            creds["session"] = {"brave_port": brave_port, "profile_dir": profile_dir,
                                "keep_open": bool(keep)}
            _save_secure(creds)  # thread-safe (uses per-account filename)

        if owned_proc and not keep:
            try:
                owned_proc.terminate()
                print("[+] closed account %d Brave (--keep-sessions to hold it)" % (idx + 1))
            except Exception:
                pass
        if not ok:
            print("[!] account %d did not complete" % (idx + 1))
        return ok, creds

    results = []
    # Run all accounts CONCURRENTLY (each its own thread + its own isolated Brave).
    max_workers = count if shared_port is None else 1
    with _cf.ThreadPoolExecutor(max_workers=max_workers) as ex:
        futures = {ex.submit(create_one, i): i for i in range(count)}
        for fut in _cf.as_completed(futures):
            idx = futures[fut]
            try:
                ok, creds = fut.result()
            except Exception as e:
                print("[!] account %d raised: %s" % (idx + 1, str(e)[:80]))
                ok, creds = False, None
            results.append((idx, ok, creds))

    results.sort(key=lambda r: r[0])
    created = [c for _, ok, c in results if ok and c]
    ok_all = len(created) == count

    if created and ok_all:
        print("\n[OK] %d FOFA account(s) created & logged in:" % count)
        for c in created:
            line = "    email: %s | user: %s | pwd: %s" % (c["email"], c["username"], c["password"])
            if keep:
                line += " | session port %s" % (c["session"]["brave_port"])
            print(line)
        sys.exit(0)
    else:
        print("\n[FAIL] not all accounts completed (%d/%d ok)" % (len(created), count))
        sys.exit(1)


def _save_secure(creds):
    """Write a per-account record (incl. session info) to the secure dir. Idempotent."""
    secure_dir = os.environ.get("FOFA_SECURE_DIR", os.path.expanduser("~/.fofa-accounts"))
    try:
        os.makedirs(secure_dir, mode=0o700, exist_ok=True)
        per = os.path.join(secure_dir, creds["email"].replace("@", "_at_") + ".json")
        with open(per, "w") as f:
            json.dump(creds, f, indent=2)
        os.chmod(per, 0o600)
        print("[+] saved email+session+creds ->", per)
    except Exception as e:
        print("[!] secure-dir save failed:", str(e)[:50])


if __name__ == "__main__":
    main()
