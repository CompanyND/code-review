# Bitbucket PR → Claude Code Review Bot

Automatický code review bot: přijme Bitbucket webhook, stáhne diff a Jira ticket, a zaposílá review jako komentáře přímo do PR.

---

## Jak to funguje — celý tok

```
Bitbucket PR (created / updated)
        │
        ▼
  /webhook/bitbucket
        │
        ├─► Ověření HMAC podpisu (volitelné)
        ├─► Deduplikace (SQLite, 15 min TTL)
        ├─► Skip branch? (release/prod → ignoruj)
        │
        ├─── paralelně ──────────────────────────────┐
        │   Stažení diffu (Bitbucket API)            │
        │   Stack verze: Angular + .NET (s cache)    │
        └────────────────────────────────────────────┘
        │
        ├─► Filtrování diffu (package-lock.json, *.min.js, …)
        ├─► Počítání změněných řádků → přes limit? → skip komentář
        │
        ├─► Jira ticket (název, popis, AC)
        ├─► Výběr modelu (Sonnet / Opus) ← viz níže
        ├─► prioritize_diff — seřadit soubory dle důležitosti, zkrátit na 400 000 znaků
        │
        ├─► Claude API (system prompt cached, user prompt s diffem)
        │
        └─► Bitbucket komentáře
              ├─ souhrnný komentář (overview, doporučení, sekce)
              └─ inline komentáře k řádkům (paralelně, max 6)
```

---

## Instalace a spuštění

### Požadavky

```
python >= 3.11
fastapi
uvicorn
httpx
```

```bash
pip install -r requirements.txt
uvicorn main:app --host 0.0.0.0 --port 8000
```

### Environment variables

| Proměnná | Povinná | Popis |
|----------|---------|-------|
| `ANTHROPIC_API_KEY` | ✅ | API klíč Anthropic |
| `BB_OAUTH_CLIENT_ID` | doporučeno | Bitbucket OAuth2 Client ID |
| `BB_OAUTH_CLIENT_SECRET` | doporučeno | Bitbucket OAuth2 Client Secret |
| `BB_TOKEN_<REPO>` | fallback | Per-repo token (např. `BB_TOKEN_MY_REPO`) |
| `BB_WEBHOOK_SECRET` | doporučeno | HMAC secret pro ověření webhooků |
| `JIRA_BASE_URL` | volitelná | Např. `https://firma.atlassian.net` |
| `JIRA_EMAIL` | volitelná | Email pro Jira API auth |
| `JIRA_API_TOKEN` | volitelná | Jira API token |

### Nastavení Bitbucket webhooku

1. Repozitář → **Settings → Webhooks → Add webhook**
2. URL: `https://tvuj-server/webhook/bitbucket`
3. Trigger: `Pull Request: Created` + `Pull Request: Updated`
4. Secret: hodnota `BB_WEBHOOK_SECRET`

---

## Výběr modelu (Sonnet vs Opus)

Bot automaticky volí mezi `claude-sonnet-4-6` (rychlý, levný) a `claude-opus-4-7` (výkonnější) podle složitosti PR.

Každý signál přidá 1 bod do skóre složitosti:

| Signál | Threshold |
|--------|-----------|
| Velký PR | > 1 000 změněných řádků |
| Mnoho souborů | > 15 souborů |
| Bezpečnostní soubory | název obsahuje `auth`, `login`, `password`, `permission`, `jwt`, `oauth`, `secret`, `encrypt` |
| Dependency / migrace | `.csproj`, `package.json`, nebo `migration` v názvu souboru |

**Skóre ≥ 2 → Opus, skóre < 2 → Sonnet.**

---

## Prioritizace diffu

Pokud je diff větší než 400 000 znaků (~100 000 tokenů), bot soubory *neukrojí náhodně* — seřadí je podle důležitosti a vezme tolik, kolik se vejde:

| Priorita | Soubory |
|----------|---------|
| 100 — nejvyšší | auth, login, password, permission, jwt, oauth, secret, encrypt |
| 90 | migration, seed, schema |
| 50 | `.cs`, `.ts`, `.py`, `.java`, `.go`, `.rb`, `.php` |
| 30 | `.html`, `.scss`, `.sass`, `.css` |
| 20 | ostatní |
| 10 — nejnižší | `*.spec.*`, `*.test.*` |

---

## Filtrování souborů

Následující soubory jsou automaticky vyloučeny z diffu i z počítání řádků (generované soubory nemá smysl reviewovat):

```
package-lock.json, yarn.lock, pnpm-lock.yaml
composer.lock, Gemfile.lock, poetry.lock
*.min.js, *.min.css
```

Vlastní seznam lze upravit v proměnné `IGNORED_FILES` v `main.py`.

---

## Skip větve

PR mířící do těchto větví jsou automaticky ignorovány (merge PR bez změn kódu):

- přesný název: `release`
- prefix: `release/…`
- suffix: `…/release`

---

## Limit velikosti PR

Pokud PR po vyfiltrování překročí **8 000 změněných řádků**, bot přidá informativní komentář a review neprovede. Doporučuje se rozdělit PR.

Limit lze změnit v `POC_MAX_LINES` v `main.py`.

---

## Detekce stack verzí (s cache)

Bot automaticky detekuje verze Angular a .NET z repozitáře a přizpůsobí prompt:

- **Angular** — čte `package.json` v rootu i podsložkách, hledá `@angular/core`
- **.NET** — hledá `.csproj` soubory, parsuje `<TargetFramework>`

Výsledky jsou **cachovány v paměti po dobu 1 hodiny** per repozitář — bot nevolá Bitbucket API při každém PR.

Na základě verze bot přizpůsobuje:
- kontext architektury (NgModules vs Standalone, .NET Framework vs .NET 8)
- performance tipy (trackBy, Signals, EF lazy loading, IHttpClientFactory…)
- security tipy (CSRF, IDOR, mass assignment, weak cryptography…)

---

## Struktura komentáře v PR

### Souhrnný komentář

```
🤖 Automatické code review (Claude AI) | Jira: ABC-123

| Změněných řádků | 342 |
| Angular verze   | 17  |
| .NET verze      | net8.0 |

---
### ✅ APPROVE

Stručný popis co PR dělá.

**Klíčové body:**
* bod 1
* bod 2

---
### 🐛 Bugy a logické chyby
…

### 🔒 Bezpečnost
…
```

### Inline komentáře

Přidány přímo k řádkům kódu, max 6 na PR. Každý obsahuje:

```
🤖 Claude - Code Review
---
🔴 CRITICAL 🐛 Bug

BLOCKER: Popis problému a navržená oprava.
```

Severity: `🔴 CRITICAL` / `🟠 MAJOR` / `🟡 minor` / `⚪ nit`

---

## Deduplikace

Bot ukládá zpracované PR do SQLite (`dedup.db`) s klíčem `workspace/repo/pr_id/commit_hash`. Pokud Bitbucket odešle stejný webhook vícekrát (retry), bot ho ignoruje. TTL záznamu je 15 minut.

---

## Health check

```
GET /health
```

Vrátí stav všech integrací, konfiguraci limitů a počet záznamů v stack cache.

---

## Nasazení (Railway / Render)

1. Forknout repo, napojit na Railway/Render
2. Nastavit environment variables (viz tabulka výše)
3. Start command: `uvicorn main:app --host 0.0.0.0 --port $PORT`
4. Nastavit Bitbucket webhook na public URL serveru
