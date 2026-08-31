# FOFA Account Automation

Fully-automated, end-to-end creation of **FOFA / NOSEC** search-engine accounts
(`en.fofa.info`, via `i.nosec.org` registration). Handles the whole lifecycle, including the
tricky bits that normally stop automation cold:

- **Disposable-email domain blocklist bypass** — FOFA rejects every common temp‑mail provider.
  This tool uses the one provider verified to work: `tempinbox.xyz` (it serves the address on its
  own fixed, FOFA‑accepted domain AND exposes a readable `/mailbox` view).
- **Cloudflare Turnstile** — auto‑solves in a headed browser, with a reload‑and‑retry loop that
  recovers from the intermittent *"Failed to verify"* state.
- **Rails honeypot** — the `#nosectest` bait field is cleared before submit.
- **Activation email** — pulled from the tempinbox `/mailbox` DOM, confirmation link clicked,
  account confirmed, then logged into the FOFA dashboard.
- **Secure archival** — every created account's credentials **and** the captured activation email
  are written to a private, chmod‑0600 per‑account file.

The tool is a single Python script (`fofa`) that drives a real Brave browser over Chromium DevTools
Protocol (CDP) with Playwright. No `computer_use`/GUI automation — fully scriptable and repeatable.

---

## How it works (the hard parts solved)

### 1. The email problem
FOFA/NOSEC validates the signup email by **domain blocklist**. We brute‑verified ~70 temp‑mail
domains against the register endpoint. **Rejected** (a sample): `westcast-systems.com`,
`emalupe.com`, `guerrillamailblock.com`, `guerrillamail.com`, `guerrillamail.de`, `grr.la`,
`maildrop.cc`, `necub.com`, `tozya.com`, `vemailgo.shop`, `tempmail.email`, `temp-mail.org`,
`tmails.net`, `getnada.com`, `mailinator.com`, `yopmail.com`, `jetable.org`, `emailondeck.com`,
`mintemail.com`, `spam4.me`, and ~31 more.

**`tempinbox.xyz` is the winner** because it uniquely combines:
- Its address is served on the **own fixed domain** `tempinbox.xyz` (most temp‑mail services rotate
  to alternate sub‑domains — `tozya.com`, `vemailgo.shop`, `thepiratebay.cloud` — that FOFA blocks).
- It has a readable inbox: a `/mailbox` session view (and an `endpoint.tempinbox.xyz` API).

> ⚠️ **Delivery subtlety** — tempinbox boxes created via the site (`#create` / `#random`) deliver the
> activation email to the browser session's **`/mailbox` DOM**, *not* to the
> `endpoint.tempinbox.xyz/messages/{email}` API (which returns `[]` for site‑created boxes). The
> tool therefore reads the confirmation token from the `/mailbox` page content.

### 2. Cloudflare Turnstile
The widget auto‑solves in a headed browser, but intermittently reports **"Failed to verify"** and
never fills `cf-turnstile-response`. `solve_turnstile()` and `do_login()` reload the page (up to ~4×)
until the token populates (~752 chars), then submit.

### 3. Login form specifics
- Field IDs are `#username` and `#password` (not `#user_email`).
- The submit is a **`button[type=submit]`** ("Login"), **not** `input[type=submit]` — clicking the
  wrong one silently no‑ops. The tool clicks it via JS.
- `#fofa_service` (Agree to FOFA Platform Service Agreement) must be checked or login stalls.
- Login returns a 303 then a transitional "Logging in..." page; success is detected by dashboard
  content ("FOFA Search Engine", "Asset Reward", "Logging in..."), not just the URL.

---

## Requirements

- **Linux** with a **headed Brave** (Chromium) — `brave-browser` on PATH at `/usr/bin/brave-browser`.
- **Python 3.14** venv with **Playwright** and a working `greenlet` C extension.
  > If your venv is missing/breaking `greenlet`, use a different one (e.g. `fofa-venv`). The tool
  > auto‑detects `/home/kali/fofa-venv/bin/python3.14` and `/home/kali/pb/bin/python3.13`.
- `DISPLAY` set (e.g. `:0`) so Brave can render head‑less‑less (a real window; needed for Turnstile).
- Network access to `i.nosec.org`, `en.fofa.info`, `www.tempinbox.xyz`, `endpoint.tempinbox.xyz`.

Install:
```bash
python3 -m venv ~/fofa-venv && ~/fofa-venv/bin/pip install playwright
~/fofa-venv/bin/python -m playwright install chromium   # or point at system Brave
chmod +x fofa && ln -sf "$PWD/fofa" ~/bin/fofa           # optional: put `fofa` on PATH
```

---

## Usage

The **recommended** invocation needs no manual browser setup — `fofa` spins up its own fresh Brave
(clean profile, auto‑chosen free port) and tears it down when done:

```bash
fofa --create                         # full flow; auto-launches a fresh Brave + cleans up
fofa --create 5                       # create 5 accounts; each gets its OWN isolated browser session
fofa --create 5 --keep-sessions       # ... and leave each account's Brave OPEN (recorded per-account)
fofa --create --brave-port 9224       # attach to an existing single Brave (shares one session)
fofa --create --out ~/fofa-accts.json # registry output file (default ~/fofa-accounts.json)
fofa --create --password 'S3cretPw!'  # override account password
fofa --create --secure-dir /path/dir  # per-account archive dir (default ~/.fofa-accounts)
```

### `--create N` = N isolated accounts, created CONCURRENTLY

`--create` takes an integer count (default 1). All N accounts are created **in parallel** — each runs
in its own thread against its **own freshly-launched Brave** (separate `--user-data-dir` profile and
its own unique CDP port, handed out by a thread-safe monotonic allocator so no two sessions ever
collide). The accounts are fully independent (no shared cookies/sessions) and are minted at the same
time rather than one-after-another. With `--keep-sessions` (alias `--keep-brave`), each account's Brave
is left running and the archive records that session's **CDP port and profile dir**, so you can
reconnect to and reuse each account's browser independently (e.g. via Playwright
`connect_over_cdp("http://127.0.0.1:<port>")`).

```
[OK] 3 FOFA account(s) created & logged in:
    email: fofagenXXXX@tempinbox.xyz | user: secpocXXXXXX | pwd: SecPass123!x | session port 9499
    email: fofagenYYYY@tempinbox.xyz | user: secpocYYYYYY | pwd: SecPass123!x | session port 9498
    email: fofagenZZZZ@tempinbox.xyz | user: secpocZZZZZZ | pwd: SecPass123!x | session port 9497
```

Each account's archive file (`~/.fofa-accounts/<email>.json`) includes a `session` object:
`{"brave_port": <port>, "profile_dir": <path>, "keep_open": true}` for reuse.

> ⚠️ On a constrained machine, spawning many headed Brave instances at once can cause transient
> nav/Turnstile timeouts. Those are retried automatically (reload + re-solve per account); each
> account's flow runs independently so one slow account doesn't block the others. Pair it with
> `--brave-port` only if you deliberately want ONE shared session (then accounts run sequentially).

### Flags

| Flag | Default | Meaning |
|------|---------|---------|
| `--create N` | `1` | number of accounts to create. Each gets its own isolated browser session. |
| `--brave-port` | *(auto)* | CDP port of an existing Brave. If omitted, a fresh Brave is started per account on a free port. |
| `--keep-sessions` / `--keep-brave` | off | leave each account's Brave open and record its CDP port + profile dir in the archive |
| `--out` | `~/fofa-accounts.json` | where the registry (list of accounts) is appended |
| `--password` | `SecPass123!x` | the FOFA account password (the tool's shared "easy" default) |
| `--secure-dir` | `~/.fofa-accounts` | per-account file archive (creds + captured email + session info) |
| `--python` | *(auto-detect)* | Python interpreter that has Playwright |

### Output

On success the tool prints the credentials and exits `0`:

```
[OK] FOFA account created & logged in:
    email: fofagenxxxx@tempinbox.xyz
    password: SecPass123!x
    username: secpocyyyy
```

It writes two things per run:

1. **Registry** (`--out`): a JSON list of every account created (appended).
2. **Secure archive** (`--secure-dir`, default `~/.fofa-accounts`, dir mode `0700`): one file per
   account named `<email>.json` (file mode `0600`) containing the creds **and the captured
   activation email** (sender, subject, date, message body) — so you always have the confirmation
   receipt, not just a login.

---

## The register/login flow (what the script does)

1. **Generate a tempinbox address** on the FOFA-accepted `tempinbox.xyz` domain: load
   `www.tempinbox.xyz`, fill `#user`, force `#domain=tempinbox.xyz` via JS, click `#create`, then
   read the active address from `/mailbox`.
2. **Open the register form** (`i.nosec.org/register?locale=en&service=…`).
3. **Solve Turnstile** (reload-retry until the token populates).
4. **Clear the honeypot** `#nosectest`, check `#agree`.
5. **Submit** (fields `nosecuser[email|password|password_confirmation|username]`). Success = redirect
   to login with *"A message with a confirmation link has been sent to your email address."*
6. **Poll `/mailbox`** for the tempinbox email from `no-reply@baimaohui.net`
   (subject "Confirmation instructions") and extract the `confirmation_token`.
7. **Confirm** via `i.nosec.org/confirmation?confirmation_token=…` → *"Your email address has been
   successfully confirmed."*
8. **Log in** (`#username`/`#password`, check `#fofa_service`, solve Turnstile, click the real
   `button[type=submit]`) and detect the `en.fofa.info` dashboard.
9. **Archive** the account creds + captured activation email.

---

## Repo layout

```
.
├── fofa                      # the CLI tool (single-file Python, executable)
├── fofa.py                   # same source (for import/editing)
├── requirements.txt          # playwright
├── docs/
│   └── FOFA-ACCOUNT-CREATION.md   # full skill/playbook, domain blocklist, form maps, pitfalls
├── README.md                 # this file
└── .gitignore
```

---

## Legal / responsible use

This is a **lab/pentest research tool** built for an authorized security‑research environment.
It automates creating FOFA search‑engine accounts, which is FOFA's own registration flow. Use it
only against accounts/plan you are entitled to operate, and comply with FOFA's Terms of Service.
The `SecPass123!x` default password is a placeholder for a shared "easy" password used across a
controlled test fleet — change it (`--password`) for anything real. **Never commit real account
credentials or API tokens.**
