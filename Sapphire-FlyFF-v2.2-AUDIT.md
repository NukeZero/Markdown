# Sapphire FlyFF v2.2 — Legacy Codebase Audit

**Repository:** `https://github.com/BuffPang/Sapphire-FlyFF-v2.2`
**Audit commit:** `b2509c55ba7d02fe699e802899c318b3c2b3069d` — "Final", 23 Jul 2026, single commit, no history
**Audit date:** 2026-10-19
**Mode:** READ-ONLY. No repository file was modified, created, or deleted. No commits, branches, or pull requests.
**Scope:** 83 PHP files (13,286 LOC), 4 `.phtml` templates, 13 JS, 6 CSS, 24 SQL scripts, 4,299 images, 1 Windows executable.

---

## 0. Methodology and limitations (read this before trusting any claim)

This audit separates **three evidence classes**. Every finding is tagged.

| Tag | Meaning |
|---|---|
| **[EXEC]** | Actually executed during this audit, with the observed output recorded in this report. |
| **[STATIC]** | Established by reading source, grepping, and correlating against the shipped `.sql` schema scripts. Not executed. |
| **[UNVERIFIED]** | Plausible but dependent on runtime/deployment facts I could not observe. Listed as a question, not a finding. |

### What was actually executed

1. **Syntax lint of all 83 PHP files** under a real PHP **8.4.26** CLI. Result: **all 83 pass `php -l` — zero parse errors.** This is the single most misleading result in the audit: *the code parses cleanly on 8.4 and still fatals at runtime* (see §6). **[EXEC]**
2. **A runtime probe of removed/changed APIs** on PHP 8.4.26 (`audit_tests/t1.php`), results reproduced verbatim in §6.1. **[EXEC]**
3. **An isolated include-resolution experiment** (`/home/user/res/`, a purpose-built mirror fixture — *not* repository code) to test whether the panel bootstrap chain breaks. It does not; result in §3.3. **[EXEC]**
4. Quantitative scans: `odbc_*` call sites, prepared-statement count, superglobal use, error-suppression count, deprecated-API inventory, byte-level file headers.

### What was NOT executed, and why it matters

- **No live SQL Server.** This sandbox has PHP 8.4 but **no `php_odbc`, no `pdo_sqlsrv`, and no MSSQL instance** (`php -m` shows only PDO core). Every database conclusion is therefore **[STATIC]**, derived from source plus the `.sql` scripts in `Anhänge/SQLs/`. I cannot confirm that any table, column, view, or stored procedure actually exists in a deployed database — only that the code assumes it and that a DDL script exists (or does not).
- **No HTTP server / no browser-driven testing.** No request was sent to this application. No authentication, purchase, donation, ban, or admin action was performed or attempted.
- **No exploitation.** No payloads were fired at the application. Severity ratings are *static risk assessments* based on sink analysis and whether the pre-conditions are reachable, not demonstrated compromises.
- **The shipped `.exe` was not executed or reverse-engineered.** It is reported as a supply-chain concern only.

Where I say "SQL injection is reachable", read it as: *the sink receives attacker-controlled data with escaping that is provably insufficient in that context*. Where I say "breaks on PHP 8.4", read it as: *the call is to a function removed in PHP 8.0 and was confirmed absent on 8.4.26*.

---

## 1. Executive summary

**Verdict: rebuild the application layer; preserve the database, the domain rules, and the assets.** Repair-in-place is not defensible, and a pure "upgrade" cannot reach the stated modernization target.

This is a 2013-era German FlyFF v15/v19 web panel (`xBlubbs`/`Chyukasame`, GPL headers, a single squashed 2026 commit). Functionally it is *ambitious*: registration, login, account panel, donation shop with two currencies, PayPal/Paysafecard/ukash/Paymentwall intake, GM and admin backends, item dispatch, ban management, tickets, rankings, vote rewards, and a launcher widget. That breadth is genuinely worth preserving as a specification. The implementation underneath it is not.

**The five findings that decide the verdict:**

1. **There is no parameterized SQL anywhere.** 1,520 `odbc_*` call sites, **0** `odbc_prepare`/`odbc_execute`/PDO. Every query is string concatenation. The two "escape" primitives (`cleanSqlInput`, `mssql_escape_string`) both reduce to *doubling single quotes*, which is correct only inside string literals and **does nothing in numeric or identifier position** — and the codebase routinely drops them into numeric position (`sites/shop.php:76`, `panels/admin/shop_categories.php:43–45`, `panels/admin/site_rules.php:24`). `panels/admin/account_maintenance.php:8` uses `$_POST['username']` with **no escaping at all**. This is architectural, not a bug list. **[STATIC]**
2. **The sanitizers are fatal on the target runtime.** `cleanSqlInput()`/`antisql()` call `get_magic_quotes_gpc()` (`includes/xinc_functions.php:5,16`), removed in PHP 8.0. Every login, registration, ticket, shop purchase, and password reset routes through them. On 8.4.26 this is a confirmed `Error: Call to undefined function`. `panels/gm/search_inventory.php:142` calls `each()`, also removed. **[EXEC]**
3. **Zero CSRF protection — combined with destructive GET endpoints.** There is not one CSRF token in the codebase. The only anti-automation device (`isSpamming()`, `includes/xinc_functions.php:560`) is a replay counter that also *mutates the session*, not a request-origin check. It gates nothing that matters: `panels/admin/index.php:29–36` truncates 33 logging tables and runs `DBCC SHRINKDATABASE` on a **plain `GET`**, and `sites/validation.php:84–135` approves donations and credits `ACCOUNT_TBL.cash` on a `GET` authenticated only by an admin session plus a salt that is committed to the repository. **[STATIC]**
4. **Authorization is hardcoded identity, not a permission model.** Site-admin rights = membership of `$_CONFIG['allg_svr_admins'] = array('chyukasame','blubbs1337','accountName')` (`includes/xinc_config.php:62`). The GM gate is an `array_key_exists()` against an `odbc_result()` character (`panels/gm/inc/header.php:11`) — a construction that works by accident of type juggling and is not a role check. No `session_regenerate_id()` exists anywhere, so login does not rotate the session ID. **[STATIC]**
5. **Committed secrets that are load-bearing for money.** `donate_adminsalt = '!dgfh$869!Al56'` (`:178`) is the *entire* authorization secret for the donation-approval URL; `admin_maintenance_pw = 'start123'` (`:249`) is a universal account-override password; `security_secretAnswerSalt = '!337'` (`:251`) and `allg_svr_salt = 'kikugalanet'` (`:51`) are the password/secret-answer salts. The ReadMe tells operators to change three of these *after* deployment — in a public repo, before deployment is when it matters. **[STATIC]**

**Why "repair" fails and "rebuild" wins.** Every one of the five above is a *cross-cutting* property. Fixing them is not a patch series; it means replacing the data-access layer (1,520 call sites), the session/auth layer (every entry point), the escaping strategy (every `echo`), and every state-changing endpoint (GET → POST + token). That is the same work as a rewrite — but with the added cost of preserving 13k lines of German copy-paste whose inconsistencies are load-bearing. Conversely, the database model, the SQL scripts, the business rules (currency mapping, ban semantics, item dispatch via `ITEM_SEND_TBL`, vote cooldowns, secret-question recovery), the image/asset set, and the page structure are all reusable and should be treated as the specification.

**Against the stated target, three of your requirements are already constraints on any design:**

- **`LOGIN_STR`: not used anywhere.** `grep -rn LOGIN_STR` over the entire repository returns **0** hits across PHP, SQL and text. The "never use `LOGIN_STR`" rule costs nothing today — and my recommendation is to keep it that way (§9.3 explains why this matters for the password column).
- **No installer**: correct — there is none, and the ReadMe is a manual IIS-era runbook. Nothing to remove.
- **No `.env`**: consistent with the target, but the current substitute (a PHP config file that *executes two database queries at include time*, `:73–74`) must not survive.

**Preserve / repair / replace / remove — headline:**

| Decision | Items |
|---|---|
| **Preserve (as spec + assets)** | Database schema and the 24 SQL scripts; `usp_createAccount` contract; MD5+salt password *compatibility*; FlyFF column names/authority letters; all 4,299 image assets; German `lang/de.php` copy; page/feature taxonomy |
| **Repair (small, worth keeping)** | Ticket state machine; ranking SQL; ban/TempPassword semantics; `send_item()` shape (rewrapped in PDO with a transaction) |
| **Replace (rewrite)** | Data access (→ PDO), auth/session, escaping/output, router, both panels, shop purchase, donation approval, config |
| **Remove** | `sites/test.php`, `panels/gm/language/test.php`, `sites/features.php` (unreachable duplicate), `panels/admin/site_configuration.php` (form saves nothing), duplicated `ajax.php:12–18` block, `Anhänge/Ruby - ACP Item List Creator.exe` |

**Effort shape (not a quote):** the reusable surface is the schema + rules + assets. The disposable surface is 100% of the request-handling code. A rebuild is *smaller in net new code* than a repair, because roughly a third of the current 13k lines is dead, duplicated, or unreachable.

---

## 2. Complete feature inventory

Status legend: **F** = appears functional by static reading; **B** = broken/never-wired (static evidence); **R** = requires runtime testing; **S** = security-defective by design.

### 2.1 Public site (`sites/`, routed through `index.php`)

| Feature | Entry point | Key files | DB objects | Status |
|---|---|---|---|---|
| Front router / page allowlist | `index.php?site=` | `index.php:144–151` | — | **F** |
| News list (top 15, categories, icons) | `?site=news` (default) | `sites/news.php` | `web_news`, `web_newscategories` | **F** / **S** (stored XSS) |
| Registration | `?site=regist` | `sites/regist.php` | `ACCOUNT_TBL`, `ACCOUNT_TBL_DETAIL`, `VALIDATE`, `usp_createAccount` | **S** (fatal on 8.4; SQLi-capable; no CSRF) |
| Login / logout | sidebar box | `includes/xinc_login.php`, `includes/box_loginbox.php` | `ACCOUNT_TBL`, `ACCOUNT_TBL_DETAIL`, `LOG_LOGIN` | **S** (fatal on 8.4) |
| Password recovery — "I know my password" | `?site=forgotpw&reset` | `sites/forgotpw.php:4–75` | `ACCOUNT_TBL`, `ACCOUNT_TBL_DETAIL` | **F** / **S** |
| Password recovery — "set a new one" | `?site=forgotpw&change` | `sites/forgotpw.php:76–147` | `VALIDATE` (type 1) | **F** / **S** |
| Email/registration validation landing | `?site=validation&key=` | `sites/validation.php:4–82` | `VALIDATE` (types 0/1/2) | **F** / **S** (no auth, no expiry) |
| Donation approval/denial (admin, via email link) | `?site=validation&key=&salt=` | `sites/validation.php:83–151` | `LOG_DONATIONS`, `ACCOUNT_TBL`, `ACCOUNT_TBL_DETAIL`, `LOG_BANS` | **S** (GET, secret in repo) |
| Account panel — overview, characters, last logins | `?site=accountpanel` | `sites/accountpanel.php:377–525` | `CHARACTER_TBL`, `GUILD_MEMBER_TBL`, `GUILD_TBL`, `LOG_LOGIN`, `LOG_LOGIN_TBL` | **R** (depends on game tables) |
| Account panel — set security info (email/secret Q/A/birthday) | `?site=accountpanel` (first run) | `sites/accountpanel.php:33–118` | `ACCOUNT_TBL_DETAIL` | **S** (validation results dropped, see §5) |
| Account panel — change email (double opt-in) | `?site=accountpanel&changeMail` | `sites/accountpanel.php:119–196` | `VALIDATE` (type 2) | **F** |
| Account panel — change password | `?site=accountpanel&changePassword` | `sites/accountpanel.php:197–265` | `ACCOUNT_TBL`, `ACCOUNT_TBL_DETAIL` | **F** / **S** |
| Account panel — donation / purchase / gift logs | `&dLogs`, `&bLogs`, `&gLogs` | `sites/accountpanel.php:266–376` | `LOG_DONATIONS`, `LOG_PURCHASES`, `LOG_GIFTS` | **F** |
| Account panel — guild rejoin | `&do=guildrejoin&name=` | `sites/accountpanel.php:379–385` | `CHARACTER_TBL` | **B** (hardcoded `20121023160610`, GET mutation) |
| Donation shop — browse (category → sub → item) | `?site=shop` | `sites/shop.php`, `includes/xinc_shop.php` | `SHOP_CATEGORIES_MAIN/_SUB`, `SHOP_ITEMLIST` | **F** / **S** |
| Donation shop — buy with DP/VP | `POST ?site=shop` | `sites/shop.php:72–157` | `ACCOUNT_TBL.cash/votepoints`, `LOG_PURCHASES`, `ITEM_SEND_TBL` | **S** (non-atomic, SQLi in numeric position) |
| Donation shop — gift to another character | `POST shopGiftItem` | `sites/shop.php:158–196` | `LOG_GIFTS`, `ITEM_SEND_TBL` | **S** |
| Shop AJAX fragment endpoint (called directly by JS) | `includes/xinc_shop.php` | `js/shop.js` | as above | **B/S** (corrupted file header, §5-M6) |
| Donation intake — PayPal / Paysafecard / ukash | `?site=donate` | `sites/donate.php` | `LOG_DONATIONS` | **R** (needs live processor) |
| Donation intake — Paymentwall pingback | `includes/xinc_paymentwall.php` | — | `LOG_DONATIONS`, `ACCOUNT_TBL` | **S** (replay double-credit) |
| Vote rewards (3 toplists) | sidebar box | `includes/box_votescript.php` | `LOG_VOTE`, `ACCOUNT_TBL.votepoints` | **F** / **S** (blocking `sleep(20)`) |
| Rankings — players / GW / guilds / votes / stats | `?site=ranking&rnk=` | `sites/ranking.php` | `CHARACTER_TBL`, `GUILD_TBL`, `LOG_VOTE`, `LOG_ITEM_TBL`, `tblCombat*` | **R** |
| Ticket system (user side) | `?site=ticketsystem` | `sites/ticketsystem.php` | `TICKET_THREADS`, `TICKET_POSTS` | **F** / **S** |
| Downloads list | `?site=download` | `sites/download.php` | `web_downloads` | **F** / **S** (stored XSS) |
| FAQ / rules / features / team / step-by-step / imprint | `?site=faqs|rules|features|team|stepbystep|impressum` | `sites/*.php` | `web_faq`, `web_rules` | **F** (`features` unreachable) |
| Error page | `?site=fehlerseite` | `sites/fehlerseite.php` | — | **F** |
| Launcher widget (separate full page) | `launcher.php` | — | `web_news`, `ACCOUNT_TBL`, `CHARACTER_TBL`, `GUILD_TBL`, `LOG_USER_CNT_TBL` | **F** / **S** (hardcoded IP) |
| Dev leftovers | `sites/test.php`, `panels/gm/language/test.php` | — | — | **B** (dead) |

### 2.2 Admin panel (`panels/admin/`, 26 PHP files, ~5,600 LOC)

| Feature | File | Notes / status |
|---|---|---|
| Login gate | `inc/header.php:12–20` | **S** — username-array authorization |
| Dashboard + DB maintenance | `index.php` | **S** — truncate/backup on `GET`; **B** — `filemtime()` on empty log name |
| Database backup | `inc/functions.inc.php:342–363` | **B** — SQL typo `TOO DISK` makes every backup fail (§5-M4) |
| Logging-DB truncate | `inc/functions.inc.php:365–423` | **S** — unconditional `TRUNCATE` of 33 tables + `DBCC SHRINKDATABASE` |
| Guild-siege log reset | `inc/functions.inc.php:425–432` | **S** — unconditional `DELETE` on `tblCombat*` |
| News + categories CRUD | `site_news.php` | **F** / **S** (stored XSS downstream) |
| Downloads CRUD | `site_downloads.php` | **F** / **S** |
| FAQ CRUD | `site_faq.php` | **F** |
| Rules CRUD | `site_rules.php` | **F** / **S** (SQLi in numeric position) |
| Partner/banner CRUD | `site_partner.php` | **R** — table `rev3_partner` has **no DDL** shipped |
| "Features" CRUD | `site_features.php` | **B** — writes into `web_faq`, same rows as FAQ |
| Static configuration form | `site_configuration.php` | **B** — form has **no POST handler at all**; saves nothing |
| Shop categories / items CRUD | `shop_categories.php`, `shop_items.php` | **F** / **S** |
| Account search / detail / ban / unban | `account_users.php` (354 L) | **F** / **S** (IDOR-shaped, GET-driven) |
| Item dispatch (full item builder) | `account_senditem.php` | **F** / **S** (free-text item name into `ITEM_SEND_TBL`) |
| Donation points credit/debit | `account_senddps.php` | **F** / **S** |
| Account "maintenance mode" (TempPassword) | `account_maintenance.php` | **S** (unescaped POST; universal password) |
| Donation logs | `donation_logs.php` | **F** |
| Purchase / gift / ban / GM logs | `list_buylogs.php`, `list_giftlogs.php`, `list_banlogs.php`, `list_gmlogs.php` | **F** |
| Misc DB queries, pushes, deletes, PMs, GS ranking, login log | `list_database.php` (640 L) | **S** (many `GET` mutations incl. `RemAccAuth`, `PrivateMessage&del`, `DeleteLog`) |
| Online players | `list_onlineplayer.php` | **F** |
| Ticket management | `ticketsys.php` | **F** |
| AJAX endpoints (58 `odbc_exec`) | `ajax.php` (856 L) | **S** (`echo $_GET['new']`, GET grants, duplicate block) |
| Item request queue | `reqitm.php` | **R** — table `web_requestitem` has **no DDL** shipped |
| Registration list | `registration.php` | **F** |

### 2.3 GM panel (`panels/gm/`, 13 PHP files, ~2,440 LOC)

| Feature | File | Notes / status |
|---|---|---|
| Login gate | `inc/header.php:11` | **S** — weak authority test, no regeneration |
| Search (account/char/IP) + ban | `search.php`, `ajax.php` | **F** / **S** (`$validateSelectArray` undefined at `ajax.php:51`; dead `'Passwort'` branches at `:57–64`) |
| Item search | `search_item.php` | **F** |
| Inventory viewer / editor | `search_inventory.php` | **B** — `each()` fatal on PHP 8 (§6.1); writes item names back to DB from DB reads |
| Mail search | `search_mails.php` | **F** |
| Trade search / trade logs | `search_trades.php`, `inc/sites/tradelogs.php` | **F** |
| Online players | `list_online.php` | **F** / **S** (raw DB → HTML) |
| Pushed characters | `list_pushedchars.php` | **R** |
| Bans / registrations / tradings | `list_bans.php`, `list_registrations.php`, `list_tradings.php` | **F** |
| GM ticket management | `ticketsys.php` | **F** / **S** (`str_replace('script','')` as XSS defence) |

### 2.4 Game integration surface

| Integration | Mechanism | Evidence |
|---|---|---|
| Account creation | `usp_createAccount` stored procedure | `Anhänge/SQLs/usp_createAccount.sql` |
| Login | direct `ACCOUNT_TBL` read + MD5 compare in PHP | `includes/xinc_login.php:19–33` |
| Password change | direct `UPDATE` of 4 columns | `sites/accountpanel.php:223–234` |
| Item delivery | `INSERT INTO ITEM_SEND_TBL` | `includes/xinc_functions.php:208–236` |
| Online state | `CHARACTER_TBL.MultiServer > 0` | `includes/xinc_config.php:73`, `launcher.php:18` |
| Ban | `ACCOUNT_TBL_DETAIL.BlockTime` (`Ymd` string) | `sites/validation.php:103`, `panels/admin/account_users.php:149` |
| GM rights | `ACCOUNT_TBL_DETAIL.m_chLoginAuthority` ∈ {F,P,Z,N,M,L} | `includes/xinc_config.php:63–69` |
| Vote rewards | `ACCOUNT_TBL.votepoints` (custom column) | `Anhänge/SQLs/ACCOUNT_TBL.sql` |
| Server presence | TCP connect | `launcher.php:21`, `includes/xinc_functions.php:551` |

---

## 3. Repository architecture and file map

### 3.1 Tree (abridged; image dirs collapsed)

```
Sapphire-FlyFF-v2.2/
├── index.php                    166 L   front controller + layout shell
├── launcher.php                  52 L   standalone launcher widget (full HTML doc)
├── LICENSE                              MIT (2026 BuffPang) — contradicts GPL headers in every file
├── ReadMe.txt                    7 KB   German IIS-era setup runbook (= the "installer")
├── README.md                      120 B  stub
├── includes/                    1,680 L  shared layer
│   ├── xinc_config.php           251 L   config + live DB queries + secrets
│   ├── xinc_functions.php        651 L   sanitizers, mail, item send, formatting, UI helpers
│   ├── xinc_login.php             64 L   login/logout logic (no output)
│   ├── xinc_detection.php         79 L   "SQL injection detection" (rewrites $_GET/$_POST, logs)
│   ├── xinc_lang.php              21 L   language file loader (?lang=, file_exists)
│   ├── xinc_shop.php             182 L   AJAX shop fragment renderer (directly requestable)
│   ├── xinc_paymentwall.php      177 L   Paymentwall pingback listener (directly requestable)
│   ├── box_loginbox.php           60 L   login box view
│   ├── box_votescript.php        108 L   vote reward logic + view
│   ├── site_boxes.phtml / site_navigation.phtml / site_footer.phtml / site_slider.phtml
│   └── security/injections_0.txt        attack log shipped with repo (1 line, 2013)
├── sites/                       2,908 L  18 page controllers (whitelisted)
├── panels/
│   ├── admin/                   5,616 L  26 pages + inc/{header,footer,navigation,functions}
│   └── gm/                      2,443 L  13 pages + inc/{...}
├── lang/de.php                   510 L   German only (no en.php despite ?lang=)
├── styles/  (6 CSS: reset, style, additional, lightbox, slider, panel style.css)
├── js/      (jQuery 1.7.2 + 6 local plugins)
├── fonts/   (5 TTF)
├── images/  (4,299 PNG, 65 JPG, 63 GIF, 1 PSD, 1 ICO)
└── Anhänge/                      German "attachments" = DB setup kit
    ├── SQLs/                     24 .sql (schema + 2 stored procedures)
    └── Ruby - ACP Item List Creator.exe   81 KB Windows binary
```

### 3.2 Bootstrap and request flow

Front-end (`index.php`):

```
session_start()                      index.php:2         ← no cookie params, no regeneration
define('access', true)               index.php:3         ← the *only* thing stopping direct
require_once xinc_config.php         index.php:4            file access to includes/ and sites/
require_once xinc_lang.php           index.php:5
require_once xinc_functions.php      index.php:6
require_once xinc_detection.php      index.php:7         ← mutates $_GET/$_POST, writes a log file
require_once xinc_login.php          index.php:8         ← performs login + DB writes on every page
  → emits <head>, CSS, jQuery
  → checkDatabaseConnection sproc (extra round trip on every request)   index.php:99–105
  → layout → sidebar boxes → router:
       site = preg_replace('/[^A-Za-z0-9]/','',$_GET['site'])          index.php:146
       if site ∈ web_allowed_sites AND file_exists(sites/site.php)      index.php:147
          OR in_array($_SESSION['user'], allg_svr_admins)               ← precedence bug
       require_once sites/site.php                                       index.php:148
```

Panel pages (`panels/admin/*.php`) repeat the bootstrap with a different order and **`include` instead of `require_once`**:

```
include('inc/header.php')                 page:1
  session_start()                         inc/header.php:3
  define('access', true)                  :6
  include ../../includes/xinc_detection.php   :7   → require_once('xinc_config.php') (run #1)
  include ../../includes/xinc_config.php      :8   → run #2  ← see §3.3
  include inc/functions.inc.php           :9
  include language/english.php            :10
  authorize or die                        :12–20
```

### 3.3 Verified bootstrap behaviour (executed)

I initially hypothesised that `xinc_detection.php`'s CWD-relative `require_once('xinc_config.php')` would fail when included from `panels/admin/` (CWD = `panels/admin`, where no such file exists). **I tested this and the hypothesis is wrong**: PHP resolves a relative `require` against the *calling file's* directory when the path is not in `include_path`, so the include is found. Good — I am not reporting a false positive. **[EXEC]**

The mirror fixture revealed a different, real defect: the header's `include()` (not `require_once`) loads the config a **second** time.

```
$ cd res/panels/admin && php index.php
PHP Warning:  Constant CFG_LOADED already defined in .../includes/xinc_config.php on line 3
detect_ok=true
cfg_executions=2          ← the same file executed twice in one request
```

Consequence in the real code: `xinc_config.php:15` calls `odbc_connect()` twice per panel request (an unclosed extra connection), and `:73–74` run two aggregate queries twice. It also re-`define()`s `ODBC_USER`/`ODBC_PASS`, which emits warnings that are then hidden by `display_errors=false`. **[STATIC + EXEC]** (finding M5)

### 3.4 Coupling and duplication

- **God file:** `includes/xinc_config.php` mixes secrets, DB connection, live queries, permissions, UI strings, and feature flags. It is included by every entry point *and* by `xinc_lang.php` and `xinc_detection.php`.
- **Views and logic are inseparable:** page controllers both compute and `echo`; there is no template layer. The 4 `.phtml` files are the only partial separation.
- **Duplication confirmed by reading:** `panels/admin/ajax.php:12–18` is a byte-identical `denyValidation` block twice; shop rendering exists in **both** `sites/shop.php` and `includes/xinc_shop.php` (~180 lines duplicated with drift); `pchart`-era leftovers in CSS comments; `getJobName()` (`admin/inc/functions.inc.php`) duplicates `getjob()` (`includes/xinc_functions.php:505`) with *different* label text ("Lord Templer" vs "Templar").
- **Two panel scaffolds are copy-paste forks** of each other (identical `header.php`/`navigation.inc.php`/`functions.inc.php` shape, diverging bugs).
- **Error handling is suppression:** 343 `@`-suppressed calls. Failures (including failed backups and failed item delivery) are indistinguishable from success at the UI.
---

## 4. Database integration and authentication flow

### 4.1 Connection layer

| Property | Value | Evidence |
|---|---|---|
| Extension | **`odbc_*`** (not `pdo_sqlsrv`, not `sqlsrv_*`, not `mssql_*`) | 1,520 `odbc_*` call sites |
| Call sites | `odbc_exec` ×529, `odbc_result` ×177, `odbc_fetch_array` ×149, `odbc_num_rows` ×114 | grep count **[EXEC]** |
| Prepared statements | **0** | `grep odbc_prepare|odbc_execute|PDO::` → 0 **[EXEC]** |
| Connection | `odbc_connect('DRIVER={SQL Server};SERVER=localhost\moon', ODBC_USER, ODBC_PASS)` | `includes/xinc_config.php:15` |
| Credentials | `ODBC_USER`/`ODBC_PASS` both `''` → Windows integrated auth assumed | `includes/xinc_config.php:13–14` |
| Driver name | `{SQL Server}` = SQL Server 2000-era ODBC driver; **not** `ODBC Driver 17/18 for SQL Server` | `:15` |
| Error handling | **none** — `odbc_connect()` return value is never checked | `:15` |
| Queries at include time | 2 aggregate queries execute whenever the config is loaded | `:73–74` |

**Consequences.**

- The driver string `DRIVER={SQL Server}` will silently fail on any modern Windows host unless the legacy driver is installed. This alone is a deployment blocker for a 2026 rebuild and must become a DSN/`sqlsrv`-style connection with an explicit driver name.
- Because the connection is never validated, a DB outage produces `false` handles. `index.php:99–105` then does `odbc_result(odbc_exec(...), 'Return')` **without** suppression on the outer call — on PHP 8 (`odbc_result(resource|false)`), a `false` handle raises a `TypeError`, so a database outage yields a **fatal error page** rather than the intended "no database connection" hint. **[STATIC]**
- `SERVER=localhost\moon` reveals the instance name; consistent with a single-host deployment.

### 4.2 Schema surface and DDL coverage

Four databases, always addressed with three-part names (`[DB].[dbo].[table]`): `ACCOUNT_DBF`, `CHARACTER_01_DBF`, `WEBSITE_DBF`, `LOGGING_01_DBF` (`includes/xinc_config.php:16–21`.

19 website-side tables ship DDL. Verified table names (note two filenames are misleading — `CATEGORIES_MAIN.sql` creates `SHOP_CATEGORIES_MAIN`): **[EXEC]**

`VALIDATE`, `LOG_DONATIONS`, `LOG_PURCHASES`, `LOG_GIFTS`, `LOG_BANS`, `LOG_ITEMS`, `LOG_LOGIN`, `LOG_SENDITEM`, `LOG_VOTE`, `SHOP_ITEMLIST`, `SHOP_CATEGORIES_MAIN`, `SHOP_CATEGORIES_SUB`, `TICKET_THREADS`, `TICKET_POSTS`, `web_news`, `web_newscategories`, `web_faq`, `web_rules`, `web_downloads`.

**Referenced by code but with no DDL shipped** (deployment will fail on these pages unless the operator already has them): `web_requestitem` (5 refs — item request queue, `panels/admin/reqitm.php`, `ajax.php:45–54`), `WEB_NICK_MESSAGE` (5 refs — `list_database.php:293–301`), `rev3_partner` (3 refs — `site_partner.php`).

Game-side tables (`CHARACTER_TBL`, `GUILD_TBL`, `GUILD_MEMBER_TBL`, `INVENTORY_TBL`, `ITEM_SEND_TBL`, `BANK_TBL`, `tblCombatInfo/JoinGuild/JoinPlayer`) correctly have no DDL — they belong to the server files. `ACCOUNT_TBL.sql` and `ACCOUNT_TBL_DETAIL.sql` are **ALTER-only** scripts that add web columns to existing game tables: `votepoints int NOT NULL DEFAULT 0` and `secretquestion smallint, secretanswer varchar(50), birthday varchar(10), latestpassword varchar(32), BanReason varchar(50), regIP varchar(39), lang varchar(3)`.

This is the single most reusable asset in the repository — and the clearest evidence for "preserve the DB, replace the app".

### 4.3 Stored procedures (signatures verified against source)

`usp_createAccount` (`Anhänge/SQLs/usp_createAccount.sql:20–58`) — **verified**, not assumed:

```sql
@account varchar(32), @pw varchar(32), @cash int = 0, @email varchar(100) = '',
@gefra smallint = 0, @geant varchar(50), @birthday varchar(10),
@ip varchar(39), @lang varchar(3)
```

Behaviour that matters for a rewrite:

- Idempotence check `if not exists (select * from ACCOUNT_TBL where account = @account)`; returns **`1`** on success, **`-1`** on error (rollback), **`0`** if the account exists (`:52,57`).
- Inserts into `ACCOUNT_TBL(account,password,isuse,member,id_no1,id_no2,realname,OldPassword,cash)` with `id_no1 = id_no2 = OldPassword = @pw` and `realname = 'P'` — **the MD5 password is copied into four columns** (`:40`).
- Inserts into `ACCOUNT_TBL_DETAIL` with `gamecode='A000'`, `tester='2'`, **`m_chLoginAuthority='F'`** (normal player), `EndTime = +10 years`, `WebTime`/`BlockTime = yesterday` (`:42`).
- **`@cash` is never passed by the website**, though the parameter exists — a latent feature (start balance) with no caller.

`checkDatabaseConnection` (`checkDatabaseConnection.sql:24–34`) returns `1` if `INFORMATION_SCHEMA.TABLES` contains a table named `web_news`, else `0`. It is **not** a connection test — it is a "is the website schema installed" test, and it is schema-unscoped (`TABLE_NAME` only, ignoring `TABLE_SCHEMA`). It runs on **every** front-end page (`index.php:99–105`).

**Critical operational detail:** both call sites test the *truthiness of `odbc_exec()`*, not the procedure's return value. `sites/regist.php:69–79` therefore reports "Account erfolgreich registriert" whenever the batch executes — **including when the procedure returned `0` (account already exists) or `-1` (rollback)**. Registration success/failure is not actually observable through this code path. **[STATIC]**

### 4.4 Password and secret storage model

| Secret | Algorithm | Storage | Evidence |
|---|---|---|---|
| Account password | `md5($salt . $password)`, salt = `'kikugalanet'` (global constant) | `ACCOUNT_TBL.password varchar(32)` **plus** `id_no1`, `id_no2`, `OldPassword` | `xinc_functions`/`xinc_login.php:31`, `regist.php:16`, `usp_createAccount.sql:40` |
| Latest password | same MD5 | `ACCOUNT_TBL_DETAIL.latestpassword varchar(32)` | `accountpanel.php:233`, `validation.php:50` |
| Secret answer | `md5($secretAnswerSalt . $answer)`, salt = `'!337'` | `ACCOUNT_TBL_DETAIL.secretanswer varchar(50)` | `regist.php:70`, `forgotpw.php:26,98` |
| Validation tokens | `random_string(32)` from `mt_rand` seeded with `mt_srand(microtime)` | `VALIDATE.[key] varchar(32)` | `xinc_functions.php:30–40`, `forgotpw.php:105` |
| Temp password | `md5($salt . 'start123')` — a **constant** | `ACCOUNT_TBL.TempPassword` ↔ `password` | `account_maintenance.php:11–22` |

Three properties are load-bearing for your "preserve FlyFF account compatibility / no destructive DB changes" constraint:

1. **`password` is `varchar(32)`.** MD5 hex is exactly 32 chars. Moving to `password_hash()` (bcrypt = 60 chars, argon2id ≈ 97) **does not fit** without a schema change. Any modern hashing plan must either add a new column (additive, non-destructive) or accept MD5 for game-login parity. This is the concrete reason the `LOGIN_STR` prohibition matters: if the client ever supplies a pre-hashed credential, the server-side scheme is not free to change.
2. **`OldPassword` + `id_no1`/`id_no2` are not dead fields** — the site deliberately rotates among them. `forgotpw.php:29–50` ("I know my password") restores `OldPassword` **as the new password**, i.e. the recovery path *reverts* to a previously used password and puts it in three columns. This is a design decision that must be re-specified, not silently carried forward.
3. **Salt is global, not per-user.** Identical passwords produce identical hashes across the whole user base.

### 4.5 Session model

```
session_start()                       index.php:2; admin/inc/header.php:3; gm/inc/header.php:3; includes/xinc_shop.php:2
$_SESSION['user']  = strtolower($account)     xinc_login.php:39; regist.php:72; validation.php:26
$_SESSION['lang']  = lang from ACCOUNT_TBL_DETAIL   xinc_login.php:40
$_SESSION['votepoints'] / ['cash']    regist.php:73–74 (dead — never read)
session_destroy() on logout, cookie not cleared     xinc_login.php:8–12
```

Verified by grep **[EXEC]**:

- **`session_regenerate_id()` — 0 occurrences.** Login does not rotate the session identifier → classic **session fixation** exposure (an attacker who can plant a cookie pre-login inherits the authenticated session).
- **No `session_set_cookie_params`, no `session.cookie_httponly`/`secure`/`samesite` set in code** — 0 occurrences. Cookie flags are entirely at the mercy of `php.ini`.
- **No `header('Location')` redirects anywhere** — navigation is via `?site=` on the current URL, `<meta refresh>` (`validation.php:34`, `ticketsystem.php:64`), and `<meta http-equiv="refresh">` (`gm/list_online.php:2`). This means every page is reachable by GET with state in the query string — which is what makes the CSRF findings below so destructive.

### 4.6 Authorization model — three tiers, all identity-based

| Tier | Rule | Evidence | Assessment |
|---|---|---|---|
| Site administrator | `in_array($_SESSION['user'], $_CONFIG['allg_svr_admins'])` — hardcoded `array('chyukasame','blubbs1337','accountName')` | `xinc_config.php:62`; `admin/inc/header.php:13`; `box_loginbox.php:22`; `index.php:144,147` | Works, but rights live in a PHP array in the repo. No roles, no audit, no revocation without a code deploy. |
| GM | authority letter test via `array_key_exists(odbc_result(...), $_CONFIG['allg_svr_rights'])` | `gm/inc/header.php:11`; `gm/ajax.php:8`; `box_loginbox.php:25` | **Fragile by construction** (see below). |
| Player | any row in `ACCOUNT_TBL` | `xinc_login.php` | — |

**The GM check analysed.** `array_key_exists($key, $array)` inspects the **keys** of `$array`. Here `$array` is `$_CONFIG['allg_svr_rights']`, whose keys are the letters `P,Z,N,M,L` (`xinc_config.php:63–69`), and `$key` is `odbc_result(...)` — the first column of the first row, i.e. a single character. So the test means: *"is the account's authority letter one of P/Z/N/M/L?"*

- `array_key_exists('F', ...)` → false → ordinary players denied. Correct.
- **`array_key_exists('', ...)` → false** — and `odbc_result` returns `''`/`false` on a missing row or failed query. So a MySQL-style "fail closed" happens only by luck of the letters chosen.
- If a future admin adds `'F' => 'User'` to `allg_svr_rights` for display purposes (a natural thing to do — `authgroup()` already maps `'F'` to `'User'` at `xinc_functions.php:482`), **every player in the game gains GM-panel access**. Two arrays in the same file already disagree about whether `F` is a right (`:63–69` vs `authgroup()`).
- `panels/gm/ajax.php:8` additionally ORs in the site-admin list, so the AJAX surface has a *different* rule from the page surface.

This is not a "weak but working" check to be preserved; it is a permission model that must be replaced with an explicit allowlist of authority letters, server-side, in one place.

### 4.7 Login sequence (traced)

```
POST loginSend
 ├─ cleanSqlInput($_POST['loginAccount'])          xinc_login.php:17   → FATAL on PHP 8.4 (get_magic_quotes_gpc)
 ├─ SELECT account,password WHERE account='<input>'   :19             → concatenated, quote-doubled
 ├─ isSpamming(timestamp, key, $_SESSION[key])      :22                → not CSRF; mutates session
 ├─ empty checks                                    :25                → "fill in all fields"
 ├─ if odbc_num_rows < 1                            :28                → distinct "no account" message → user enumeration
 ├─ if md5(salt.pw) != odbc_result(password)        :31                → loose !=, non-constant-time
 ├─ else $_SESSION['user'] = strtolower(account)    :39                → NO session_regenerate_id
 │        $_SESSION['lang'] = SELECT lang …         :40
 │        INSERT INTO LOG_LOGIN(account, ip, host, browser, timestamp) :42–56
 │             host    = gethostbyaddr($_SERVER['REMOTE_HOST'])        → REMOTE_HOST usually unset
 │             browser = $_SERVER['HTTP_USER_AGENT']  raw, unescaped   → SQL break on apostrophe in UA
 └─ (no rate limit, no lockout, no failed-attempt log)
```

Two secondary defects worth flagging for the rewrite: the `HTTP_USER_AGENT` is concatenated raw into an `INSERT` (`:53`) so a user agent containing `'` breaks the query — and because it is `@`-suppressed, the failure is invisible; and `gethostbyaddr()` on an unset `REMOTE_HOST` triggers a warning plus a potentially blocking reverse lookup. Both disappear naturally when the layer becomes parameterized with `$_SERVER` fallbacks.

### 4.8 Registration sequence (traced)

`POST registSubmit` → validators (`sites/regist.php`):

| Check | Line | Note |
|---|---|---|
| Account exists (`ACCOUNT_TBL`) | 18 | raw concatenation, **no escaping at all** here |
| Account pending (`VALIDATE type=0`) | 19 | same |
| Honeypot (`isSpamming`) | 26 | no `isset()` guards; missing fields → PHP 8 warnings |
| Required fields | 29 | — |
| Account 3–16 alnum | 38–43 | `preg_replace` comparison |
| Password 6–16 | 44 | also enforced client-side via `maxlength` mismatch (form has no maxlength) |
| Secret answer alnum+space | 50 | — |
| Email regex | 56 | `^[a-zA-Z0-9-_.]+@…\.[a-zA-Z]{2,4}$` — rejects modern TLDs (`+`, >4 chars) |
| Disposable-domain blocklist | 59 | `web_mail_provider` array, 100+ hardcoded domains (`xinc_config.php:157–163`) |
| Birthday parseable | 62 | `strtotime()` only — accepts `now`, `+1 day`, etc. |

Then `EXEC usp_createAccount …` with account, `md5(salt.pw)`, email, `$_POST['registSecretQuestion']` (**not validated numeric**), `md5(secretAnswerSalt . answer)`, `date()`, `REMOTE_ADDR`, lang (`:69–71`). On any `odbc_exec` truthiness → `$_SESSION['user']` is set and the user is **auto-logged-in** (`:72–74`).

> `$provider = preg_replace("^(.*?)@(.*?)^", "$2", $email);` (`:12`) uses `^` as the delimiter, which is legal but bizarre; the pattern then matches the whole string non-greedily and captures the domain. It "works" (confirmed in isolation: `a@b.c` → `b.c` **[EXEC]**) but is unmaintainable and duplicated verbatim at `sites/accountpanel.php:43`.

### 4.9 Token / recovery flow

```
forgotpw&change  → validates email + secret question + secret answer hash
                 → generates key=random_string() and an 8-char password
                 → INSERT VALIDATE(type=1, fieldAlfa=md5(salt.newpassword))     forgotpw.php:105–122
                 → mails link index.php?site=validation&key=<key>                :123

validation&key=  → SELECT * FROM VALIDATE WHERE key='<key>'   (no salt, NO AUTH, NO EXPIRY)  validation.php:6–10
                 → type 0: EXEC usp_createAccount (values read from VALIDATE row) + DELETE   :29–32
                 → type 1: UPDATE ACCOUNT_TBL SET id_no2=password, id_no1=<new>, password=<new>  :42–54
                 → type 2: UPDATE ACCOUNT_TBL_DETAIL SET email=<new>                          :66–69
```

`VALIDATE.timestamp` is written on every insert and **never read**. Keys are therefore valid forever until consumed. A key in a GET URL is also consumed by any mail-security scanner or chat-link preview that fetches the URL — the single-use token is burned before the human clicks it.

### 4.10 Database assumptions that must be verified before reuse

| Assumption | Where | Risk if wrong |
|---|---|---|
| `ACCOUNT_TBL` has `votepoints`, `TempPassword`, `id_no1`, `id_no2`, `OldPassword`, `realname`, `member`, `cash` | throughout | silent `@`-suppressed failures |
| `ACCOUNT_TBL_DETAIL` has `latestpassword`, `secretquestion`, `secretanswer`, `birthday`, `regIP`, `lang` | `ACCOUNT_TBL_DETAIL.sql` | only added by the ALTER script |
| `CHARACTER_TBL` uses `MultiServer > 0` for "online" and `isblock` ∈ {`F`,`D`} | `xinc_config.php:73`, many | rankings/online lists wrong |
| `GUILD_MEMBER_TBL` links `m_idPlayer`→`m_idGuild`; `GUILD_TBL.m_idGuild` | `accountpanel.php:401–402` | guild column blank |
| `ITEM_SEND_TBL` accepts `(m_idPlayer, serverindex='01', Item_Name, Item_count, idSender='0000000', m_nAbilityOption, m_bItemResist, m_nResistAbilityOption, nPiercedSize)` | `xinc_functions.php:212–226`, `account_senditem.php:10–24` | item delivery silently fails |
| `serverindex '01'` is the right channel for **all** deliveries | `send_item()` | items land on the wrong server |
| `LOGGING_01_DBF.LOG_LOGIN_TBL` has `remoteIP`, `Start_Time` | `accountpanel.php:436–442` | "last in-game login" breaks |
| `m_chLoginAuthority` letters `P/Z/N/M/L` mean staff | `xinc_config.php:63–69`, `authgroup()` | see §4.6 |
| DB collation is case-**insensitive** | all `WHERE account = …` | duplicate accounts differing only by case |
| `LOG_DONATIONS.[key] varchar(32)` can hold Paymentwall `ref` values | `LOG_DONATIONS.sql` | PK truncation/insert failure |
| `ITEM_SEND_TBL.idSender` `'0000000'` is accepted by the server build | `send_item()` | delivery rejected by game |

---

## 5. Security findings

**Severity distribution: 8 Critical · 8 High · 8 Medium · 6 Low · 5 Informational.**

All ratings are **static** risk assessments (§0). "Reachable" describes code paths, not demonstrated exploitation.

| ID | Sev | Finding | Primary evidence |
|---|---|---|---|
| SEC-01 | **Critical** | No parameterized SQL anywhere; escaping is quote-doubling and fails outside string literals | 529 `odbc_exec`, 0 prepares; `shop.php:76`; `shop_categories.php:43–45`; `site_rules.php:24` |
| SEC-02 | **Critical** | Unescaped POST concatenated into SQL | `account_maintenance.php:8` |
| SEC-03 | **Critical** | Second-order SQLi: values read from `VALIDATE`, concatenated into `EXEC`/`UPDATE` | `validation.php:30`; `admin/ajax.php:34`; `gm/ajax.php:54` |
| SEC-04 | **Critical** | No CSRF anywhere **and** destructive/credit-granting actions on `GET` | `admin/index.php:29–36`; `admin/ajax.php:45–98`; `list_database.php:152–157`; `validation.php:84–135` |
| SEC-05 | **Critical** | Load-bearing secrets committed; universal account-override password | `xinc_config.php:51,178,249,251`; `account_maintenance.php:11–22` |
| SEC-06 | **Critical** | Identity-based authorization; flawed GM authority test; no session regeneration (fixation) | `xinc_config.php:62–69`; `gm/inc/header.php:11`; `xinc_login.php:39` |
| SEC-07 | **Critical** | Money/credit integrity: pingback replay double-credit, loose signature compare, non-atomic purchases | `xinc_paymentwall.php:104,108–156`; `shop.php:128–150` |
| SEC-08 | **Critical** | Stored/reflected XSS on public pages and in both panels | `news.php:22,26`; `download.php:33–36`; `xinc_shop.php:72,112–138`; `admin/ajax.php:464,705,851`; `gm/search.php:6` |
| SEC-09 | **High** | MD5 passwords, global salt, password replicated to 3 columns; recovery restores an *old* password | `xinc_config.php:51`; `usp_createAccount.sql:40`; `forgotpw.php:29–50` |
| SEC-10 | **High** | No login throttling/lockout; user enumeration; non-constant-time compare | `xinc_login.php:28–33` |
| SEC-11 | **High** | `isSpamming()` is not a CSRF control; its PRNG is predictable and reused for reset passwords | `xinc_functions.php:30–40,560–570` |
| SEC-12 | **High** | Validation tokens in URLs, never expire, no auth for types 0–2 | `validation.php:4–82`; `VALIDATE.sql` |
| SEC-13 | **High** | "Injection detection" rewrites request data and writes attacker-controlled text to a web-readable log | `xinc_detection.php:46–78` |
| SEC-14 | **High** | Ban audit trail fabricated: `byadmin` hardcoded `'marvin'` | `validation.php:105–119` |
| SEC-15 | **High** | `includes/security/` must be world-writable and is web-readable; no `.htaccess`/deny rules shipped | `ReadMe.txt:28–31`; no `.htaccess` in repo |
| SEC-16 | **High** | Item dispatch accepts free-text item names with no validation or ownership check | `account_senditem.php:5–24`; `send_item()` |
| SEC-17 | Medium | No security headers; no cookie flags in code | 0 hits for `X-Frame-Options`, `Content-Security-Policy`, `SameSite`, `cookie_httponly` |
| SEC-18 | Medium | Pervasive `@` suppression hides security-relevant failures | 343 suppressed calls; `display_errors=false` |
| SEC-19 | Medium | Ticket: `ip varchar(15)` truncation, forgeable close token, no CSRF, unescaped body echo | `TICKET_THREADS.sql`; `ticketsystem.php:102` |
| SEC-20 | Medium | Shop: affordability class computed from wrong variable; arbitrary recipient; no quantity bound | `shop.php:325` (`$strItemlist` vs `$strItemArray`) |
| SEC-21 | Medium | Config executes twice per panel request (double connection, redefined constants) | `admin/inc/header.php:7–8` **[EXEC]** |
| SEC-22 | Medium | Unexamined Windows binary shipped in-repo | `Anhänge/Ruby - ACP Item List Creator.exe` (81 KB) |
| SEC-23 | Medium | jQuery 1.7.2 (2012) with `.live()` usage; `.live()` removed in jQuery 1.9 | `js/jquery-1.7.2.min.js`; `js/shop.js:5–15` |
| SEC-24 | Medium | Column-width/collation assumptions that break integrity at scale | `LOG_DONATIONS.key varchar(32)`; `TICKET_*.ip varchar(15)`; `VALIDATE.account varchar(255)` vs 16 |
| SEC-25 | Low | `$_SESSION['user']` echoed raw into panel headers | `admin/inc/header.php:35,37`; `gm/inc/header.php:34,36` |
| SEC-26 | Low | Reflected XSS in search `value=` attributes | `gm/search.php:6`; `admin/site_search.php:6` |
| SEC-27 | Low | Router precedence bug + array-parameter `TypeError` | `index.php:146–148` |
| SEC-28 | Low | `mail()` header construction with config-derived `From`; no CRLF filtering on address | `xinc_functions.php:43–205` |
| SEC-29 | Low | Output corruption: `0x12` byte before `<?php`, leading newline in `regist.php` | `xinc_shop.php` offset 0 **[EXEC]** |
| SEC-30 | Low | `gethostbyaddr($_SERVER['REMOTE_HOST'])` — unset key, unbounded reverse DNS | `xinc_login.php:52` |
| SEC-31 | Info | GPL headers in every file vs MIT `LICENSE` | `LICENSE` vs `xinc_config.php:1–6` |
| SEC-32 | Info | Attacker IP permanently committed in shipped log | `includes/security/injections_0.txt:1` |
| SEC-33 | Info | Third-party/hardcoded infrastructure references | `launcher.php:21,48`; `admin/inc/functions.inc.php:21–26` |
| SEC-34 | Info | Dead/unreachable code and pages | `sites/test.php`; `sites/features.php`; `site_configuration.php`; `admin/ajax.php:12–18` |
| SEC-35 | Info | No unified audit log for admin actions | scattered `LOG_BANS`/`LOG_SENDITEM`/`LOG_PURCHASES` |

### 5.1 Critical findings — detail

#### SEC-01 — SQL injection by architecture
**Evidence.** 529 `odbc_exec()` calls, zero parameterized. Two helpers exist and both are inadequate:

```php
// includes/xinc_functions.php:4–10
function antisql($s) { if(get_magic_quotes_gpc()){$s = stripslashes($s);} 
    return str_replace(array("'", '"'), array("'", '""'), $s); }      // " → "" is NOT SQL Server escaping
// includes/xinc_functions.php:11–26  cleanSqlInput(): strip_tags + same quote doubling
// panels/admin/inc/functions.inc.php:168–171 / panels/gm/inc/functions.inc.php:2–5
function mssql_escape_string($string){ return str_replace("'", "''", $string); }
```

`str_replace("'", "''")` **is** valid T-SQL escaping — but only *inside* a quoted literal. The codebase applies it in unquoted numeric/identifier positions, where it is a no-op:

```php
// sites/shop.php:76        ← purchase handler, logged-in user, numeric context, no is_numeric()
WHERE [mid] = ' . cleanSqlInput($_POST['iid']) . ' AND [category] = ' . cleanSqlInput($_POST['sid'])
// panels/admin/shop_categories.php:43–45
SET [mid] = ' . mssql_escape_string($_POST['mid']) . ',  [active] = ' . mssql_escape_string($_POST['active'])
// panels/admin/site_rules.php:24
WHERE [id] = '.mssql_escape_string($_POST['edit_faqid'])
```

A payload such as `iid=1 OR 1=1` or `sid=0 OR [mid]=1` needs **no SQL keywords**, so the blacklist in `xinc_detection.php:46` (which removes `select `, `union `, etc.) does not obstruct it.

**Risk.** Authenticated (shop) and admin-context database access with the application's own credentials — which are SQL-Server *dbowner*-level by default in this deployment style. Data theft (accounts, MD5 hashes, emails), balance manipulation, and DDL capability.

**Remedy.** This is not fixable by better escaping. Every one of the 529 sites becomes PDO with bound parameters; numeric values get explicit `PDO::PARAM_INT` casts; identifiers are allowlisted, never interpolated. Delete `antisql`, `cleanSqlInput`, `mssql_escape_string` rather than "improving" them.

#### SEC-02 — Unescaped POST into SQL
**Evidence.** `panels/admin/account_maintenance.php:8`

```php
$retval = odbc_exec($odbc_connect, 'SELECT * FROM [ACCOUNT_TBL] WHERE [account] = \''.$_POST['username'].'\'');
```

Lines 11, 12, 20 and 22 in the same file *do* escape; line 8 does not. **Risk:** admin-context injection (the panel is authenticated, but GM-panel→admin escalation is not the only path — CSRF chaining makes it reachable without the password). **Remedy:** parameterize; never hand-roll.

#### SEC-03 — Second-order injection through `VALIDATE`
**Evidence.** Values are written to `VALIDATE` (escaped, or not, depending on the flow), then read back and concatenated **without re-escaping**:

```php
// sites/validation.php:30
EXEC ...usp_createAccount @account='$account', @pw='$pass', @email='$email',
     @gefra='$secretQuestion', @geant='$secretAnswer', @birthday='$birthday', @ip='$ip', ...
// panels/admin/ajax.php:34   (same, from the admin side)
// panels/gm/ajax.php:54      (same; also $validateSelectArray undefined at :51)
```

`regist.php`, `forgotpw.php` and `accountpanel.php` all insert into `VALIDATE` with different escaping discipline. **Risk:** a stored payload that fires later, from a different trust context (including admin). **Remedy:** parameterize the `EXEC` call; treat `VALIDATE` as untrusted on read; delete the row in the same transaction as the effect.

#### SEC-04 — No CSRF, plus state-changing GET
**Evidence.** Zero CSRF tokens in the codebase. `isSpamming()` (`xinc_functions.php:560–570`) sets `$_SESSION['inputCheckKey'] = $strPostKey` and requires the posted key to differ from the *previous* session value — a resubmission guard that **also mutates the session**, making it hostile to any future token design. It prevents neither cross-origin submission nor link-clicking.

Critical endpoints accept `GET` with no token:

| Action | Evidence | Effect |
|---|---|---|
| Truncate 33 logging tables + `DBCC SHRINKDATABASE (0)` | `panels/admin/index.php:29–32` → `ClearLoggingDatabase()` | destroys log data |
| Back up all DBs | `panels/admin/index.php:33–36` | (fails anyway — SEC/`TOO DISK`) |
| Approve/deny donation and credit `cash` | `sites/validation.php:84–135` | **grants currency** |
| GM grant/revoke, ban UI, FAQ edit, psc credit/debit, item-request accept (which calls `send_item`) | `panels/admin/ajax.php:41–98`; `panels/gm/ajax.php:18–70` | grants items/rights/currency |
| Grant/remove account & character authority | `panels/admin/list_database.php:152–157` | privilege change |
| Delete private message | `list_database.php:297` | data loss |
| Close ticket | `sites/ticketsystem.php:102` | state change |

**Risk.** A single `<img src="…/panels/admin/index.php?do=delete_logging">` on any page visited by a logged-in admin destroys the logging database. A crafted `?site=validation&key=…&salt=…` link (salt is in the repo, SEC-05) credits arbitrary currency. **Remedy:** POST-only for all mutations; per-session, per-form CSRF tokens validated before side effects; no state changes on GET; `SameSite=Lax` as defence in depth.

#### SEC-05 — Committed, load-bearing secrets
**Evidence** (`includes/xinc_config.php`):

| Line | Key | Value | Role |
|---|---|---|---|
| 51 | `allg_svr_salt` | `kikugalanet` | password salt |
| 178 | `donate_adminsalt` | `!dgfh$869!Al56` | **the only secret** gating donation approval URLs |
| 249 | `admin_maintenance_pw` | `start123` | universal account-override password |
| 251 | `security_secretAnswerSalt` | `!337` | secret-answer salt |
| 175–177 | Paymentwall keys | `XXXX…` placeholders | (inert, correctly blank) |

The salt is embedded in admin notification emails (`xinc_functions.php:199–200`) as a plain query parameter, so it also leaks via mail archives, proxies and referrers. `ReadMe.txt:23–26` instructs operators to rotate three of these — *after* the repo was public.

The maintenance password is worse than a leak: `account_maintenance.php:11–22` sets any account's `password` to `md5($salt.'start123')` and stashes the real one in `TempPassword`. Anyone who can read this repository (or guess the default) knows the credential that opens **every account placed in maintenance mode**.

**Remedy.** Secrets move to a filesystem-include config *outside* the webroot and outside version control (still no `.env`, per your constraint — e.g. `includes/config.local.php` git-ignored, or environment-specific include). Rotation is mandatory, not advisory. Replace the "maintenance password" concept with an explicit, logged impersonation feature or remove it.

#### SEC-06 — Identity-based authorization, flawed GM gate, no session rotation
Covered in §4.5–4.6. Additional **evidence** for fixation: `session_start()` at `index.php:2` runs before any authentication, and `$_SESSION['user']` is assigned at `xinc_login.php:39` with **no** `session_regenerate_id()`. Logout (`xinc_login.php:8–12`) calls `session_destroy()` but leaves the cookie in place.

**Remedy.** One authorization service, server-side, keyed on an explicit allowlist of authority letters read from `ACCOUNT_TBL_DETAIL.m_chLoginAuthority` **plus** a separate `administrators` table or explicit config list; `session_regenerate_id(true)` at login and privilege change; cookie flags set in code; idle + absolute session timeout.

#### SEC-07 — Money and credit integrity
**(a) Paymentwall pingback can double-credit.** `includes/xinc_paymentwall.php:134–156` runs, as one batch:

```sql
UPDATE ACCOUNT_TBL SET cash = cash + <credits> WHERE account = '<userId>'
INSERT INTO LOG_DONATIONS ([key], …) VALUES ('<refId>', …)
```

`LOG_DONATIONS.[key]` is the **primary key** (`LOG_DONATIONS.sql`). On a retry with the same `ref` — which payment providers do routinely — the `UPDATE` succeeds, the `INSERT` violates the PK, and the batch is not wrapped in a transaction with `XACT_ABORT`. Net effect: **cash credited again on every delivery attempt**. Additionally `$credits` and `$userId` are concatenated unescaped (`:108,135`), and the signature comparison is `==` (`:104`) rather than `hash_equals()`, which is loose-typed.

**(b) Signature version is attacker-selected.** `if (empty($sign_version) || $sign_version <= 1)` (`:74`) lets the caller choose the weaker v1 signature over four parameters; v2+ signs all of `$_GET` (`:87–90`).

**(c) Shop purchase is non-atomic.** `sites/shop.php:128–150`:

```php
@odbc_exec($conn, $strUpdateBalance)              // 1. debit
AND @odbc_exec($conn, "INSERT INTO LOG_PURCHASES…")// 2. audit
AND send_item($charId, $itemId, $count)            // 3. deliver
```

No transaction, no `WHERE cash >= price` guard, no idempotency key. Concurrent requests can both pass the balance check (`:120`) before either debit lands → **double-spend**; and if step 3 fails, step 1 has already debited with no rollback (item lost, money gone).

**Remedy.** All credit/debit operations inside `BEGIN TRAN`/`COMMIT` with `XACT_ABORT ON`; idempotency keyed on the provider reference (upsert-or-noop, never double-apply); `hash_equals()` for signatures; force the highest supported signature version; debit with `UPDATE … WHERE cash >= :price` and check `rowCount()`.

#### SEC-08 — Stored and reflected XSS
Public, unauthenticated-visible sinks:

| Sink | Line | Source of data |
|---|---|---|
| News title / body (raw `nl2br`, no escaping) | `sites/news.php:22,26` | admin news editor → fires for **every visitor** |
| Download title / description / `href` | `sites/download.php:33–36` | admin; `javascript:` URL possible in `link` |
| Shop item name / icon / effects / description | `sites/shop.php:248–275`, `includes/xinc_shop.php:72–138` | admin shop editor |
| Ticket title/body in list view | `sites/ticketsystem.php:135+` | any registered user |
| GM online list (raw DB → HTML) | `panels/gm/list_online.php:32–35` | character names from game DB |
| Search field `value="…"` | `panels/gm/search.php:6`, `panels/admin/site_search.php:6` | reflected from `$_GET['search']` |
| `echo $_GET['new']` | `panels/admin/ajax.php:464,705,851` | **reflected, admin session** |
| `echo '<h2>Logs - '.$_GET['char'].'</h2>'` | `panels/admin/list_gmlogs.php:29` | reflected |
| `echo '<b>'.$_GET['account'].'</b>'` | `panels/admin/ajax.php:78`, `panels/gm/ajax.php:23` | reflected |

The only output encoder is `cleanHtmlOutput()` = `htmlspecialchars(str_ireplace('<script','', $x))` — a blacklist followed by an encoder **without `ENT_QUOTES` or a charset argument** (`xinc_functions.php:27–29`). It is used in exactly **25** places across 13k lines. `<img onerror>`, `<svg onload>`, and attribute breakouts (`" onmouseover=…`) all pass.

**Risk.** Session riding of admins/GMs (the panels have no CSRF tokens, so XSS ⇒ full administrative action), credential-harvest overlays, and — via reflected XSS in `ajax.php` — a direct path from a malicious link to admin actions.

**Remedy.** Context-aware output encoding at every sink (`htmlspecialchars($v, ENT_QUOTES|ENT_SUBSTITUTE, 'UTF-8')`), a strict CSP, and — because these sinks are stored — an allowlist HTML sanitizer *and* server-side re-encoding on read, never trust-on-write.

### 5.2 High findings — condensed

| ID | Risk | Remedy |
|---|---|---|
| **SEC-09** MD5 + global salt; password duplicated to `id_no1`/`id_no2`/`OldPassword` (`usp_createAccount.sql:40`); "I know my password" recovery sets the **old** password back (`forgotpw.php:40–50`); `latestpassword` stored in clear-MD5 (`accountpanel.php:233`) | Offline cracking of the whole user base from one dump; recovery silently weakens credentials | Additive `password_hash` column with rehash-on-login (respect `varchar(32)` constraint, §4.4); stop replicating to 3 columns; redesign recovery as a time-boxed token, never a revert-to-old |
| **SEC-10** No throttling, no lockout, no failed-login logging; distinct "no account" vs "wrong password" messages (`xinc_login.php:29,32`); `!=` on digests (`:31`) | Credential stuffing; account enumeration | Constant response, generic error, per-IP+per-account backoff, `hash_equals` |
| **SEC-11** `random_string()` seeds `mt_srand((double)microtime()*1000000)` (`xinc_functions.php:33`) and produces validation keys **and** the 8-char replacement password (`forgotpw.php:105–106`) | Predictable tokens ⇒ password reset hijack; `isSpamming` mistaken for CSRF | `random_bytes()`/`bin2hex`; separate CSRF tokens; remove `mt_srand` |
| **SEC-12** Tokens in URL, no expiry (timestamp never read), types 0–2 require no authentication | Account creation/password-change/mail-change by whoever holds a link; mail scanners burn the token | Hash tokens at rest, TTL ≤ 30 min, POST confirmation, single-use with explicit consumption after success |
| **SEC-13** `xinc_detection.php` **rewrites** `$_GET`/`$_POST` (deleting substrings like `select `) and appends attacker text unescaped to `includes/security/injections_*.txt` (`:57–78`), with `fopen()` on a path derived from a possibly empty filename (`:48–50`) | Silent data corruption ("Johnny Drop Table" → "Johnny"); log forging with CRLF; file handle churn on every request | Delete the "detection" layer; log structured, escaped events out-of-webroot; validate inputs positively |
| **SEC-14** `LOG_BANS` rows written with `byadmin` hardcoded `'marvin'` (`validation.php:116`); before/after timestamps from `mktime(...)` (`:114–115`) | Audit trail is worthless for dispute/abuse review | Pass the authenticated actor; store both `BlockTime` snapshots as readers |
| **SEC-15** `ReadMe.txt:28–31` requires `includes/security/` to be writable by the web user; the `.txt` logs are then served by Apache (`panels/admin/index.php:76` links them) and no `.htaccess`/`web.config` ships to deny `includes/` or `panels/` | Attack-log disclosure incl. internal DB names; writable directory in webroot; panels reachable directly | Move logs outside webroot; ship deny rules; disable directory listing |
| **SEC-16** Item dispatch takes a free-text `itemname` and numeric strings, no existence/ownership checks (`account_senditem.php:5–24`); `send_item()` hardwires `serverindex '01'` and `idSender '0000000'` (`xinc_functions.php:212–226`) | Any admin can mint arbitrary items; wrong-server deliveries; no reconciliation | Validate item ID against `LOG_ITEMS`/`propItem`; single transactional dispatch with audit row; make server/channel explicit |

### 5.3 Medium / Low / Informational — notes worth acting on

- **SEC-21 (config executed twice)** — measured, see §3.3. Fix by `require_once` in the panel headers; it also removes the duplicate `odbc_connect()`.
- **SEC-24 (widths)** — `TICKET_THREADS.ip varchar(15)` cannot hold IPv6; `LOG_DONATIONS.[key] varchar(32)` versus provider reference lengths; `VALIDATE.account varchar(255)` while accounts are ≤16. All three are cheap to fix *now*, while a schema change is still additive.
- **SEC-27 (router)** — `index.php:147` binds as `(in_array(...) AND file_exists(...)) OR in_array(...)`. For admins the OR short-circuits the allowlist, so **any** `sites/<alnum>.php` loads (including `test.php`, `features.php`). Non-admins are unaffected. `$_GET['site']` as an array reaches `preg_replace()` → `TypeError` on PHP 8.
- **SEC-29 (byte corruption)** — verified at byte level: `includes/xinc_shop.php` begins `12 3c 3f 70 68 70` (a `0x12` control byte before `<?php`). That byte is echoed to the client before `session_start()` runs, so on a request with output buffering disabled PHP emits *"Cannot modify header information — headers already sent"* and the byte prepends the AJAX fragment. `sites/regist.php` begins with `\n` for the same class of reason. **[EXEC]**
- **SEC-31 (licence)** — every file carries a GPL-2.0 header; `LICENSE` is MIT © 2026 BuffPang. Resolve before redistributing.
- **SEC-34 (dead code)** — `sites/features.php` exists but is absent from `web_allowed_sites`, so it is unreachable; `panels/admin/site_configuration.php` renders a 79-line form with **no `isset($_POST…)` handler at all**; `panels/gm/language/test.php` is 0 bytes; `admin/ajax.php:12–18` is duplicated verbatim.

### 5.4 Functional bugs that are not security issues (but will be inherited by any repair)

| Bug | Evidence | Impact |
|---|---|---|
| Backup SQL says **`TOO DISK =`** | `admin/inc/functions.inc.php:352` | every backup fails; failure is reported honestly per database but the feature is dead |
| `$validateSelectArray` should be `$validateArray` | `gm/ajax.php:51` | undefined variable → IP always empty |
| `VALDIDATE.type` compared to `'Passwort'` / `'Mailänderung'` strings, but the column is `smallint` | `gm/ajax.php:57,64` | both branches are unreachable dead code |
| Shop affordability uses `$strItemlist` instead of `$strItemArray` | `sites/shop.php:325` | wrong "buyable" styling |
| Validation errors appended to `$strOutputErrorArray` instead of `$strSetSecErrorArray` | `sites/accountpanel.php:59,62,139` | invalid email/domain **passes** and is written to the DB |
| `filemtime('' . '.txt')` when the log dir is empty | `admin/index.php:76` | warning + broken link |
| `filter_var`-less birthday (`strtotime`) | `regist.php:62` | accepts `now`, `+1 day` |
| Duplicate `denyValidation` block | `admin/ajax.php:12–18` | harmless, symptomatic |
| `getJobName()` vs `getjob()` disagree on class names | `admin/inc/functions.inc.php:320–340` vs `xinc_functions.php:505–546` | player/GM UIs label jobs differently |

---

## 6. PHP 8.4 compatibility findings

### 6.1 What I executed

All 83 files pass `php -l` on **PHP 8.4.26** — **zero parse errors**. **[EXEC]** That is precisely why static linting is insufficient here: the failures are *runtime* `Error`s, not syntax.

Probe results (PHP 8.4.26, verbatim):

| Expression | Result on 8.4.26 | Used at |
|---|---|---|
| `get_magic_quotes_gpc()` | `Error: Call to undefined function get_magic_quotes_gpc()` | `xinc_functions.php:5`, `:16` |
| `each([1,2])` | `Error: Call to undefined function each()` | `panels/gm/search_inventory.php:142` |
| `utf8_encode('x')` | `OK: x` (deprecated, `E_DEPRECATED`) | 16 sites (§6.3) |
| `utf8_decode('x')` | `OK: x` (deprecated) | same |
| `strlen(['a'])` | `TypeError: … must be of type string, array given` | any `strlen($_GET[...])` |
| `strtolower(['a'])` | `TypeError: …` | `xinc_detection.php:56,69` |
| `mktime(0,0,0,m,d+3,Y)` (6 args) | `OK: 20261022` | `validation.php:103,115` |
| `mktime(…10 args)` | `ArgumentCountError: expects at most 6` | *not present* in repo — checked, clean |
| `mysql_connect` / `mssql_connect` / `odbc_result_all` | do not exist | **not used** — good |
| curly-brace string offsets (`$s{0}`) | *not present* — checked, clean | — |

### 6.2 Fatal call sites (blockers)

| # | File:line | Removal | Triggers when |
|---|---|---|---|
| **F1** | `includes/xinc_functions.php:5` (`antisql()`) | `get_magic_quotes_gpc()` — removed in **PHP 8.0** | called by `box_loginbox.php:25`, `validation.php` flows, etc. |
| **F2** | `includes/xinc_functions.php:16` (`cleanSqlInput()`) | same | **every** login, registration, ticket, shop purchase, password reset, admin form |
| **F3** | `panels/gm/search_inventory.php:142` | `each()` — removed in **PHP 8.0** | opening the GM inventory viewer; `while(list(, $v) = each($arr))` |

F2 is the decisive one: `cleanSqlInput()` is the *primary* input path for the whole application (`sites/regist.php:7–15`, `xinc_login.php:17–18`, `sites/shop.php:76`, `accountpanel.php` ×many, `ticketsystem.php:55–57`, `validation.php:9`). On PHP 8.4 the site cannot process a login or a registration at all. Both functions must be reimplemented (and, per SEC-01, not reimplemented as escaping helpers at all).

### 6.3 Deprecated (works on 8.4, breaks on 9.0) — 16 call sites

`utf8_encode`/`utf8_decode` (deprecated in 8.2, scheduled for removal):

| File | Lines |
|---|---|
| `sites/ranking.php` | 77, 138, 139, 204, 207, 261 |
| `panels/gm/search_inventory.php` | 118, 121 |
| `panels/gm/inc/functions.inc.php` | 48 |
| `panels/gm/list_online.php` | 61 |
| `panels/gm/search_mails.php` | 29, 30 |
| `panels/admin/list_gmlogs.php` | 65 |
| `panels/gm/search.php` | 43 (`utf8_decode`) |

These exist because the game DB stores Windows-1252/latin1-encoded character names. The correct fix is **not** `mb_convert_encoding()` sprinkled at each call site, but one explicit conversion boundary at the data-access layer: connect with the right client charset, or convert once per result set.

### 6.4 Warnings / `TypeError`s that will flood logs and break behaviour

| Class | Evidence | Effect |
|---|---|---|
| Undefined array key → `E_WARNING` (was `E_NOTICE`) | `index.php:34,136,144,145`; `$_GET['do']` at `admin/index.php:29,33`; `is_numeric($_GET['sid'])` at `shop.php:19`; hundreds more | noise; hidden by `display_errors=false` + `E_ALL ^ E_NOTICE` (`xinc_config.php:7–8`) |
| Array-to-string `TypeError` | `xinc_detection.php:56,69`; `cleanSqlInput()` consumers; `strlen($_POST[...])` | `?site[]=x` style input → fatal, and `xinc_detection` is the *first* thing that touches it |
| `odbc_result(false, …)` on failed query | `index.php:99–105` (unsuppressed outer call); dozens of `@`-suppressed siblings | DB outage ⇒ `TypeError` page instead of a hint |
| Undefined variable | `gm/ajax.php:51` (`$validateSelectArray`); `shop.php:325` (`$strItemlist`) | wrong data written/displayed |
| `str_ireplace('<script')` as XSS defence | `xinc_functions.php:28` | not an 8.4 issue; still not a control |
| `E_STRICT`/`E_NOTICE` masking | `xinc_config.php:8` | deprecations stay invisible during migration |

### 6.5 What is *not* a problem (verified so you don't over-scope)

- No `mysql_*`, no `mssql_*`, no `ereg`/`split`, no `create_function`, no `money_format`, no `FILTER_SANITIZE_STRING`, no short tags `<?`, no curly-brace offsets, no `mktime` >6 args. **[EXEC]**
- `odbc_*` itself is **not** deprecated — but `php_odbc` must be enabled and a modern driver installed (§9.7). It is *absent* from this sandbox, which is why no DB path could be exercised.

### 6.6 Compatibility verdict

PHP 8.4 readiness of the current code: **0% functional.** It parses, and then dies at the first user interaction. A "make it run" patch is ~5 lines (reimplement two helpers, replace `each()`); the reason not to ship that patch is §5.
---

## 7. Functional vs. requiring runtime testing

No feature in this repository was executed against a database. The distinction below is therefore **"reads as coherent and self-consistent"** (F) versus **"cannot be judged without a live SQL Server and game data"** (R). Only two things were proven to *not* work (PHP 8.4 fatals) and one to be *unwired* (the configuration form).

### 7.1 Appears functional by static reading (F)

| Feature | Why it is credible |
|---|---|
| News list + categories | straightforward `TOP 15` join; matches DDL |
| FAQ / rules rendering | simple selects; matches DDL |
| Downloads list | matches DDL (`web_downloads`) |
| Ranking — players / guilds / GW / votes / stats | joins are coherent (`GUILD_MEMBER_TBL`↔`GUILD_TBL`, `tblCombatJoinGuild`); `TOP n` is `is_numeric`-gated |
| Vote box (3 toplists) | cooldown keyed on `(ip, account, toplist, timestamp)`, matches `LOG_VOTE` DDL |
| Ticket creation / listing / user-close | state machine (`status 0–4`) is consistent across user + GM + admin views |
| Account panel — logs (donation/purchase/gift) | three independent selects, all matching DDL |
| Change password / change email (double opt-in) | coherent `VALIDATE` type 2 flow |
| Admin news / FAQ / rules / downloads / shop CRUD | standard INSERT/UPDATE with (inadequate but present) escaping |
| Admin account search / ban / unban | `BlockTime` `Ymd` semantics used consistently with `validation.php` and login checks |
| GM search / mail search / trade search | parameter shapes match the game schema names |
| Launcher widget | six independent aggregates + `fsockopen` status |
| Language loader | `preg_replace` to alnum + `file_exists` before `require` — genuinely safe against traversal |

### 7.2 Requires runtime testing before any judgement (R)

| Feature | What must be verified |
|---|---|
| Login (even after the 8.4 fix) | whether `md5(salt.pw)` matches what the **client** sends; whether `ACCOUNT_TBL` collation is case-insensitive; whether `isuse='T'` matters |
| Registration + `usp_createAccount` | procedure present? `@gefra smallint` accepts the posted string? `EndTime +10 years` accepted by the server build? |
| Item delivery (`ITEM_SEND_TBL`) | exact column set per server build (`m_nAbilityOption`, `nPiercedSize`…), correct `serverindex` for multi-channel, whether the game server consumes `idSender='0000000'` |
| Shop purchase / gift | price columns (`priceDP`/`priceVP`) vs the unused `currency` column; whether VP purchases are meant to be character-bound (`$blnItemCharacterBound` is set but never used — `shop.php:87,99`) |
| Donation intake (PayPal/Paysafecard/ukash) | whether `LOG_DONATIONS` inserts align with `state` conventions; IPN/callback handling |
| Paymentwall | whether the widget is in use at all (`donate_paymentwall_enabled=false`); secret/sig version |
| Ban propagation | whether editing `BlockTime` alone locks the account (the game reads it at login?) |
| `m_chLoginAuthority` grant/revoke | whether changing the letter requires a relog/reconnect |
| Rankings | performance of the correlated subqueries in `ranking.php:158–177` on a real DB |
| `checkDatabaseConnection` | whether it returns 1 in a real deployment (table `web_news` exists) — if it returns 0, every page shows a "no DB connection" hint |
| Admin backup | **known broken** (`TOO DISK`) — needs a live DB to confirm the path/permissions model too |
| Guild rejoin | hardcoded timestamp `20121023160610` — does it still mean "rejoined"? |
| `TempPassword` maintenance flow | whether the game honours `TempPassword` |
| Ticket online-state / `list_online.php` | `MultiServer > 0` semantics per build |
| `LOG_ITEMS` item catalogue | the ReadMe's Ruby exporter is Windows-only, requires game headers — the catalogue is likely absent |

### 7.3 Proven non-functional

| Feature | Proof |
|---|---|
| Login, registration, tickets, shop, password reset, admin forms | fatal `Error` on `get_magic_quotes_gpc()` at PHP ≥8.0 **[EXEC]** |
| GM inventory viewer | fatal `Error` on `each()` **[EXEC]** |
| Admin "configuration" page | form has no handler — saves nothing **[STATIC]** |
| Database backup | `TOO DISK` typo → always fails **[STATIC]** |
| `sites/features.php` | not in `web_allowed_sites` → unreachable **[STATIC]** |
| GM `acceptValidation` type-1/type-2 branches | compare `smallint` to German strings → dead **[STATIC]** |
| Registration success reporting | tests `odbc_exec` truthiness, not the sproc's `1/0/-1` **[STATIC]** |

---

## 8. Upgrade-versus-rebuild decision

### 8.1 Options scored against evidence

Dimensions are weighted by consequence for the stated target (PHP 8.4 / PDO / MSSQL / no framework).

| Dimension | **A. Repair in place** | **B. Partial rewrite (keep framework)** | **C. Rebuild app layer, keep DB + rules + assets** |
|---|---|---|---|
| PHP 8.4 readiness | Achievable (3 call sites) but every other 8.x hazard remains (§6.4) | Same as A | Achieved by construction |
| SQL injection elimination | Requires touching **529** `odbc_exec` sites individually | Same 529 sites, just inside the old style | Single PDO layer; sinks cannot concatenate |
| CSRF | Must retrofit every endpoint; no token plumbing exists | Same | Router-level POST+token middleware |
| Output encoding | 13k lines of `echo` with string concatenation; 25 encoder calls today | Same | Template helper enforced at the boundary |
| Session/auth | Bolt-on; `isSpamming` already mutates the session and fights tokens | Same | Replaced |
| Preserve FlyFF account compat | Yes | Yes | **Yes** — schema untouched (additive only) |
| Preserve DB structures | Yes | Yes | **Yes** |
| No destructive DB changes | Yes | Yes | **Yes** |
| "No `.env` / no installer" | Yes | Yes | Yes (file-include config outside webroot) |
| Code volume to *own* afterwards | 13k lines of inherited style | ~9k lines (delete dead code) | ~4–5k lines of new code + reused SQL/assets |
| Risk of inheriting hidden bugs | **High** (343 silent suppressions) | High | Low–medium (bugs are re-specified, not copied) |
| Time to a *safe* production state | Long — security is never "done" incrementally here | Long | Comparable or shorter, with a clear end state |
| Fit to your modernization target | Poor | Medium | **Direct** |

### 8.2 Decision

**Choose C: rebuild the application layer as a thin plain-PHP application; retain the database schema, the stored-procedure contract, the business rules, the German copy and the entire asset set.**

Evidence that drives it:

1. **The defects are structural, not local.** No parameterization, no CSRF, no output encoding, no session rotation, identity-based authorization. Each is a *cross-cutting* property; adding it to 13k lines of view-and-query spaghetti costs more than writing the request layer once.
2. **The unreusable part is the request-handling layer; the reusable part is everything else.** Schema + 24 SQL scripts + sproc signatures + `LOG_ITEMS` catalogue + 4,299 images + German `de.php` + the page taxonomy survive intact. That is exactly the boundary C draws.
3. **Repair cannot reach the target stack anyway.** "PDO SQL Server where practical" + 529 `odbc_exec` sites + `DRIVER={SQL Server}` means the data layer is being replaced regardless. Once the data layer goes, the escaping/encoding assumptions that leak through every view go with it.
4. **A repair would preserve a false sense of security.** `xinc_detection.php` (an "injection detector"), `isSpamming()` (called "antispam"), and `cleanHtmlOutput()` (an XSS filter) all *look* like controls and are not. Carrying them forward is worse than having nothing.
5. **Rebuild is not "start from zero".** The audit output is itself the specification: 30 features, 19 DDL scripts, 2 stored procedures, 12 game-integration contracts (§4.10), and a page/IA map. Rebuilding those from scratch by guesswork would be riskier than reimplementing them against this document.

### 8.3 Rejected alternatives

- **A — Repair in place** is rejected because it leaves SEC-01/04/05/06/08 open for the duration of the work and does not end in a coherent security model. It is defensible *only* if the site is taken offline during repair, which is equivalent to C in disruption.
- **B — Partial rewrite of the "framework" (config/functions/login) while keeping page bodies** is the tempting middle. It fails on the shop, admin AJAX, and both panels: those pages are 60–70% of the LOC and are where the injection and CSRF sinks concentrate. You would end up rewriting them anyway, with a half-migrated codebase in the meantime.

### 8.4 What would change this recommendation

State any of these and I will re-cost the plan:

1. The site must stay **live and unchanged** for months (favours targeted containment: WAF rules, panel IP-allowlisting, disable the shop).
2. There is a hard requirement to preserve **exact current HTML/CSS** (`styles/style.css` is bespoke and tied to the sprite layout) — then C must include a template-inheritance step rather than a markup rewrite (already assumed in §9.2).
3. The FlyFF build's `ACCOUNT_TBL.password` semantics differ from `md5(salt+password)` (e.g. the client sends a pre-hash) — this changes SEC-09 and the whole auth design and must be confirmed first (§10.3 Q1).

---

## 9. Phased modernization plan

Constraints honoured throughout: **PHP 8.4 + Apache**, **MSSQL**, **PDO SQL Server**, **plain PHP + JS/jQuery + Bootstrap + Font Awesome**, **no Laravel/Node/`.env`**, **no installer**, **preserve `ACCOUNT_TBL`/game DB compatibility**, **never `LOGIN_STR` in new or modified functionality**, **no second account system**, **no assumed columns/params/permissions**, **no destructive DB changes**.

### 9.1 Phase 0 — Freeze, verify, snapshot (read-only)

| Item | Detail |
|---|---|
| Deliverables | DB schema snapshot (read-only `INFORMATION_SCHEMA` + `sys.columns` export); evidence checklist for §4.10; PHP 8.4 runtime profile of the *old* code on a scratch host for reference |
| Must verify before any code | Q1–Q8 in §10.3 (password algorithm, `item_send` columns, `MultiServer`, collation, `LOG_ITEMS` presence, `TempPassword`, `web_requestitem`, `rev3_partner`) |
| Exit criteria | every §4.10 assumption is either confirmed or marked "must change"; no production writes |
| Risk | none (read-only) |

### 9.2 Phase 1 — Foundation (the only phase that blocks everything else)

| Item | Detail |
|---|---|
| Deliverables | `bootstrap.php`; file-include config **outside** webroot (no `.env`); PDO SQL Server factory (with `sqlsrv`/ODBC fallback chosen once, verified); request helper (`input()` with casts); response/template helper with **context-aware escaping**; session service (`session_regenerate_id`, cookie flags, idle timeout); CSRF middleware; router (`GET` reads / `POST` mutations, allowlist); audit logger |
| Explicitly deleted | `antisql()`, `cleanSqlInput()`, `cleanHtmlOutput()`, `xinc_detection.php`, `isSpamming()`, `mssql_escape_string()`, `@`-suppression |
| Exit criteria | a "hello world" page + one parameterized query + login/logout with CSRF and rotation, all on 8.4 with **zero** deprecations and `E_ALL` |
| Effort | ~15% of total |

### 9.3 Phase 2 — Public read-only surfaces

News, FAQ, rules, downloads, team, imprint, step-by-step, rankings, ticket *view*, launcher widget, vote box (reward write deferred to Phase 4).

- Preserve markup/IA; port German copy from `lang/de.php` unchanged.
- Rankings: replace `utf8_encode` with a single conversion boundary; add `TOP` clamping and stable ordering.
- **Password note:** nothing in this phase touches credentials. Keep `LOGIN_STR` absent; when Phase 3 lands, the hash design must satisfy §4.4's `varchar(32)` constraint — the recommended shape is an additive `password_v2` column (bcrypt/argon2) with **rehash-on-successful-login** and MD5 verification retained for as long as game compatibility requires, never `LOGIN_STR`.
- Exit criteria: all public pages render with `E_ALL`, no `@`, no inline SQL outside the repository layer.

### 9.4 Phase 3 — Identity and account

Registration (via `usp_createAccount`, parameterized), login, logout, password change, email change, secret question/answer, validation tokens, account panel.

| Requirement | Implementation decision |
|---|---|
| Preserve FlyFF accounts | no new account table; `ACCOUNT_TBL` remains the identity source |
| Password compatibility | verify against `md5(salt.pw)`; optional additive rehash column only |
| Tokens | `random_bytes(32)`, stored hashed, TTL, POST confirmation, consumed after success — never in a GET that a mail scanner can trip |
| Recovery | no more "restore `OldPassword`"; issue a time-boxed reset token |
| Exit criteria | full auth matrix tested (§10.2); no enumeration; lockout works; session rotates |

### 9.5 Phase 4 — Commerce

Shop browse/buy/gift, donation intake, Paymentwall pingback, vote rewards, DP/VP ledgers.

- **All** balance mutations in one transaction with `XACT_ABORT ON`, `UPDATE … WHERE balance >= @price`, `rowCount()` check.
- Idempotency keys for every external credit (Paymentwall `ref`, PSP reference).
- `hash_equals()` for signatures; force the strongest signature version.
- Item dispatch via a single validated `DispatchItem` service (item ID existence check, explicit server/channel, audit row).
- Exit criteria: replay tests, concurrency tests, double-spend tests, refund/chargeback path.

### 9.6 Phase 5 — GM and admin panels

Rebuild behind a real authorization service:

- Roles resolved server-side from `m_chLoginAuthority` **plus** an explicit administrators allowlist (no array in a repo file; config-include outside VCS).
- Every mutation: POST + CSRF + audit row (actor, target, before/after, IP, timestamp).
- Destructive maintenance (truncate logging DB, backup) becomes: POST + typed confirmation + dry-run report + optional scheduled task instead of a GET.
- Backup fixed (`BACKUP DATABASE … TO DISK`), with path validation and surfaced errors.
- Exit criteria: an audit row exists for 100% of state changes; no GET-mutating endpoints remain (`grep`-enforced in CI).

### 9.7 Phase 6 — Hardening, cutover, decommission

- `.htaccess`/vhost: deny `includes/`, logs outside webroot, `display_errors=off` + `log_errors=on`, `session.cookie_*` flags, security headers, TLS.
- Remove `sites/test.php`, `sites/features.php`, `site_configuration.php`, the `.exe`, the shipped attack log.
- Dependency refresh: jQuery 3.x (replace `.live()` with `.on()` — `js/shop.js:5–15`), Bootstrap 5, Font Awesome 6, self-hosted fonts (the Google Fonts URL at `index.php:25` is long dead).
- Resolve the GPL-vs-MIT licence contradiction.
- **Deployment requirements to document:** Apache 2.4 + PHP 8.4 (`php_odbc` **or** `pdo_sqlsrv`), Microsoft ODBC Driver 18 for SQL Server, four databases, the 19 DDL scripts + 2 ALTER scripts + 2 sprocs, a service account with least privilege (`db_datareader`/`db_datawriter` + `EXECUTE` where truly needed — **not** `db_owner`), write access for logs outside the webroot, SMTP, and outbound HTTPS to the payment/toplist providers.
- Exit criteria: the old tree is retired; a rollback path (previous release + DB snapshot) is written down.

### 9.8 Sequencing rationale

Phase 1 before everything (nothing else can be built safely on the old foundation). Phase 2 before 3 so the read paths stabilise while auth is designed. Phase 4 after 3 because commerce requires identity. Panels last because they depend on every service built earlier and are the largest single LOC block (8,060 lines combined).

---

## 10. Testing requirements and unresolved questions

### 10.1 Test environment requirements

| Requirement | Reason |
|---|---|
| A **restored copy** of the production DB (never production) | `usp_createAccount`, `ITEM_SEND_TBL`, `MultiServer` semantics cannot be faked safely |
| Game-server test instance | to prove item delivery and ban semantics actually land |
| Fixed fixture set | one account, one banned account, one GM, one admin, 2 characters, 1 guild, 1 pending donation, 1 open ticket |
| PHP 8.4 with `E_ALL` and `display_errors=on` in test | deprecations must fail the build, not scroll by |
| Log capture | prove no fatal, no deprecation, no `@`-hidden error in the happy path |

### 10.2 Test matrix (minimum)

**Authentication** — login success/failure; unknown account vs wrong password produce identical responses; lockout after N attempts; session ID changes on login (`session_regenerate_id`); logout invalidates server-side session and cookie; session fixation attempt fails; CSRF token required and single-use.

**Registration** — valid/invalid account lengths and characters; duplicate account (in `ACCOUNT_TBL` and pending in `VALIDATE`); email format incl. `+` and long TLDs; disposable-domain blocklist; birthday validation (reject `now`, `+1 day`); **sproc return `0`/`-1` must produce a failure message**; no auto-login before validation if verification is enabled.

**Tokens/recovery** — expiry enforced; single use; a second GET fails cleanly; token hashed at rest; secret-answer comparison constant-time; replaced old-password-revert behaviour.

**Shop** — buy with DP and VP; insufficient funds rejected **server-side** on the selected currency only; concurrent double-spend (parallel requests) yields exactly one debit and one item; failure of `send_item` rolls back the debit; gift to a non-existent character rejected; item price/quantity taken from DB, never from POST.

**Donations** — approve/deny require POST + CSRF + admin role; replay of the same provider reference credits once; chargeback reverses once; `LOG_DONATIONS.state` transitions valid; no `'marvin'`.

**Admin/GM authorization** — player cannot reach GM or admin (including `ajax.php`); GM cannot reach admin; **adding `'F'` to the rights array must not grant access** (regression test for SEC-06); authority change audited.

**SQL injection** — automated payload suite (`' OR 1=1--`, `1 OR 1=1`, `'; EXEC xp_cmdshell--`, UTF-8/`N'` variants) against every parameter of every endpoint, asserting no 500/`odbc` errors and no unintended rows.

**XSS** — stored payloads in news, downloads, shop item name/effects, ticket title/body, character name; reflected payloads in search fields; CSP violations logged.

**Destructive-action safety** — truncate/backup unreachable by GET; typed confirmation; dry-run output; audit row.

**Regression/parity** — every preserved feature produces the same *data* as the old system (compare rendered values, not markup) for a fixed fixture set.

**Performance** — rankings and `list_database.php` (640 lines, 42 queries) under realistic row counts; N+1 patterns such as `accountpanel.php:255–256` (query per row).

### 10.3 Unresolved questions (all block or shape implementation)

1. **Password algorithm — confirm exactly.** Is the game client sending plaintext over an encrypted channel, with the server comparing `md5(salt+password)`? Or is a pre-hash involved? This determines SEC-09 and whether a `password_v2` column is viable.
2. **`LOGIN_STR`: what is it in *your* environment?** It appears nowhere in this repository. If it exists in the wider codebase/server config, its role must be documented before the auth design is frozen — otherwise "never use it" cannot be implemented deliberately.
3. **Which server build / files version?** FlyFF v15 vs v19 changes `ITEM_SEND_TBL` columns, `MultiServer`, and authority letters. The repo claims both ("v15" in the task, `v19` in the SEO tags at `xinc_config.php:156`).
4. **Is the Paymentwall path live?** `donate_paymentwall_enabled = false` (`:174`). If unused, drop the listener and its replay bug rather than porting it.
5. **Are the donation intake paths (PayPal/Paysafecard/ukash) actually in use?** If donations are manual-only, the whole approval flow can be radically simplified.
6. **`web_requestitem`, `WEB_NICK_MESSAGE`, `rev3_partner`** — do these tables exist in production? No DDL ships.
7. **`logdata` catalogue (`LOG_ITEMS`)** — is the table populated? The exporter is a Windows-only `.exe` requiring game headers.
8. **Multi-channel servers** — is `serverindex '01'` correct, or are there several (`allg_svr_channel` defines two at `:52–61`)?
9. **DB collation** — case-sensitive or insensitive? Affects duplicate-account logic and every `WHERE account =`.
10. **Who are the real administrators?** `allg_svr_admins` lists three names in the repo — are they current?
11. **Is the GM panel in production use?** It is fatally broken on PHP 8 today; if unused, it is a candidate for removal rather than rebuild.
12. **Legal** — GPL headers vs MIT `LICENSE`; the imprint data is placeholder (`Musterstraße 1`) which is not compliant for a German-facing site.
13. **`TempPassword` maintenance flow** — keep (rebuild safely) or remove?
14. **Branding/markup fidelity** — must the visual output stay pixel-identical (affects Phase 2 effort materially)?
15. **Existing data volumes** — row counts for `LOG_ITEM_TBL`, `LOG_LOGIN`, `LOG_VOTE` (drives indexing and pagination).
16. **Mail** — SMTP relay available? `mail()` is used with hand-built headers throughout.
17. **Hosting** — Windows/IIS today; will Apache/PHP 8.4 run on Windows (same host, same driver) or is this moving to Linux (then `pdo_sqlsrv`/ODBC 18 and case-sensitivity of file names matter — note `Anhänge/` has a non-ASCII directory name and `images/icons/ranking/*.PNG` mixed-case extensions, which break on Linux)?
18. **Uptime** — may the old site be taken down during the rebuild, or must both run in parallel?

---

## Appendix A — Metrics (measured)

| Metric | Value |
|---|---|
| PHP files | 83 |
| PHP LOC | 13,286 |
| LOC: `panels/admin` / `panels/gm` / `sites` / `includes` / `lang` | 5,616 / 2,443 / 2,908 / 1,680 / 510 |
| PHP files by area | 26 admin, 13 GM, 18 sites, 9 includes, 1 lang, 2 root |
| `php -l` failures on 8.4.26 | **0 of 83** **[EXEC]** |
| Runtime fatals on 8.4.26 | **3 call sites** (`get_magic_quotes_gpc` ×2, `each()` ×1) **[EXEC]** |
| `odbc_exec` call sites | 529 |
| Prepared statements | 0 |
| `@`-suppressed calls | 343 |
| Files touching `$_GET`/`$_POST` | 53 |
| `htmlspecialchars`/`cleanHtmlOutput` call sites | 25 |
| Deprecated `utf8_*` call sites | 16 |
| SQL scripts | 24 (19 CREATE TABLE, 2 sprocs, 2 ALTER, 1 connection check) |
| DDL files vs tables created | 19 files / 19 tables (2 filenames differ from the table they create) |
| Images | 4,299 PNG, 65 JPG, 63 GIF, 1 ICO, 1 PSD |
| JS / CSS / fonts | 13 / 6 / 5 |
| Commit history | 1 commit ("Final", 2026-07-23) |

## Appendix B — Dependency and deployment inventory

| Dependency | Current | Target |
|---|---|---|
| PHP | 5.x-era code, **runtime-dead on 8.x** | 8.4 |
| DB extension | `odbc_*` (`php_odbc`) | `pdo_sqlsrv` (preferred) or `pdo_odbc` with ODBC Driver 18 |
| DB driver | `DRIVER={SQL Server}` (2000-era) | `ODBC Driver 18 for SQL Server` |
| Web server | Apache/IIS (per ReadMe) | Apache 2.4 |
| SQL Server | 4 DBs, `db_owner`-ish rights implied | same DBs, least-privilege service login |
| jQuery | **1.7.2** (2012) | 3.x (rewrite `.live()`) |
| Lightbox / jCarouselLite / ddaccordion / tooltip | vendored 2012 scripts | Bootstrap 5 modals / carousel |
| Fonts | Google Fonts URL (`index.php:25`) + 5 local TTF | self-hosted |
| External services | PayPal, Paysafecard, ukash, Paymentwall, 3 toplists, TeamSpeak, `wallapi.com`, Facebook | keep only what is used; all need HTTPS + secrets outside VCS |
| Composer / npm | **none** (vendored libraries) | still none required; if Composer is added, vendor it deliberately |
| `.env` / installer | absent (matches target) | absent |
| Binary artefacts | `Anhänge/Ruby - ACP Item List Creator.exe` | remove |

## Appendix C — Checks that came back clean (so they are not re-investigated)

Verified by grep/`php -l` and **not** issues: no `mysql_*`, no `mssql_*`, no `ereg`/`split`, no `create_function`, no `money_format`, no `FILTER_SANITIZE_STRING`, no short open tags, no curly-brace string offsets, no `mktime()` with >6 arguments, no `extract()`/`parse_str()`, no `assert()`-with-string, no `eval()`, no dynamic `include` of user-controlled *paths* (the `site` parameter is regex-restricted to `[A-Za-z0-9]` at `index.php:146` and `lang` to the same at `xinc_lang.php:8` — so **no LFI**), no writable file uploads, no `unserialize()` of user data, and **no `LOGIN_STR` anywhere**.

## Appendix D — Artefacts produced by this audit

| Path | Contents |
|---|---|
| `Sapphire-FlyFF-v2.2-AUDIT.md` | this report |
| `findings.csv` | 35 findings, machine-readable, for issue-tracker import |
| `audit_tests/t1.php` | the PHP 8.4 runtime probe (reproducible) |
| `res/` | the include-resolution mirror fixture (§3.3) — scratch, not repository code |
| `repo/` | pristine clone at `b2509c5`; unmodified |

---

**No repository file was created, modified, or deleted. No commit, branch, or pull request was made. The clone at `repo/` is pristine (`git status` clean).**

**Awaiting your explicit approval before any change is made.** On approval, the first action would be Phase 0 (§9.1) — read-only verification of the §4.10 database assumptions and answers to the §10.3 questions — not code changes.
