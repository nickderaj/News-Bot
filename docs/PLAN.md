# News-Bot — build plan

A single-user news reader for the owner's own subscriptions (Financial Times and
Bloomberg). It signs in to both and collects new articles into a private store.
The articles can be read in a phone-first web app, the owner can ask questions
about them, and any article can be played aloud in xAI's **Zagan** voice. Clips
are only generated when the owner asks for one, and each is generated at most
once.

Status: planning (2026-10-04). Nothing below is built yet. The first pull request
sets up the quality gate (see [DEVELOPMENT.md](DEVELOPMENT.md)), copied from
hermes-trainer, and every later change goes through that gate.

## 1. Goals and non-goals

| Goals | Non-goals |
| --- | --- |
| Collect FT and Bloomberg articles the owner is entitled to read, with no manual steps after a one-time sign-in | Getting past a paywall the owner doesn't pay for, sharing articles, or publishing them anywhere |
| Read them in a clean, fast reader on the phone | A general RSS reader for arbitrary sites (the source adapter interface leaves room for that later) |
| Ask questions about the articles, with answers that cite them | Market data, prices or charts |
| Listen to a whole article in the Zagan voice, generated on demand and cached forever | Generating audio ahead of time, in bulk, or for anything the owner didn't ask to hear |

## 2. Decisions

| # | Decision |
| --- | --- |
| D1 | **Same shape as hermes-trainer.** Python 3.13 + FastAPI + SQLite on the server; React + Vite + TypeScript PWA on the client; served over the tailnet with `tailscale serve`; the Tailscale identity header is the login. Nothing is exposed to the internet. |
| D2 | **Strict gate from the first commit.** It is the same gate as hermes-trainer, with two extra policy rules: xAI keys count as secrets, and databases, audio files and browser captures (`.db`, `.mp3`, `.har`, …) are rejected. Every change after the bootstrap lands through a PR with the required checks and the owner's review. |
| D3 | **Signing in is a real browser session, done by a person once.** Playwright drives Chromium with a *persistent profile* per source, stored in the private data directory. The owner signs in once with `newsbot login ft` / `newsbot login bloomberg`, which opens a visible browser on the host's desktop. That handles SSO, 2FA and any captcha. The app never stores the FT or Bloomberg passwords. |
| D4 | **Collect broadly in these sections:** macro/economics, markets, bonds and rates, commodities, crypto and breaking news, from both sources (table in §2.1). The aim is volume. |
| D5 | **Discover cheaply, fetch carefully.** New article URLs come from public RSS feeds where they exist, and from section pages read in the signed-in browser where they don't. Bloomberg's feeds are thin, so most Bloomberg discovery uses section pages. Article pages are opened one at a time, at least 8 s apart per site. Each run fetches at most 60 articles per source, every 20 minutes from 06:00 to 23:00 and hourly overnight. A 429 response stops that source for the run and doubles its pause (honouring `Retry-After`). Each URL is fetched once; "updated" stories are re-fetched at most once per 6 hours while they are under 48 hours old. |
| D6 | **Never fight bot defences.** If a page shows a login wall, a captcha or a "press and hold" check, the run stops for that source and marks its session **needs attention**. The app shows a banner (and later sends a push) asking the owner to run `newsbot login` again. There is no automatic captcha solving or fingerprint spoofing. |
| D7 | **Parse structured data first.** Each source adapter reads the article from the page's embedded JSON (JSON-LD `NewsArticle`, Next.js page data) and falls back to DOM selectors. Its output is a source-neutral `ArticleDraft` (headline, standfirst, byline, section, published/updated times, ordered body blocks). Body HTML is re-rendered from those blocks through an allow-list sanitiser, never stored as raw page HTML. |
| D8 | **SQLite holds everything**, under a private data directory set in `deploy/local.env`: articles, an FTS5 index, fetch runs, clips metadata, playback progress. Clip audio is stored as files beside the database, named by content hash. Both are backed up nightly, as hermes-trainer does. |
| D9 | **Voice is xAI TTS, Zagan, on request only.** `POST https://api.x.ai/v1/tts` with `voice_id: "zagan"` and `language: "en"`, MP3 at 24 kHz / 128 kbps. A clip is keyed by **(article, narration-text hash, voice, speed, format)**. "Listen" plays the stored clip when one exists, and otherwise generates it and stores it first. There is no prefetching, batching or automatic regeneration. |
| D10 | **The cache is strict, so the paid API is called at most once per key.** The rules are pure domain functions marked `# coverage-critical`. A per-key lock means two taps (or two devices) make one request. A failed request is recorded and not retried automatically. A daily character budget (default 300 000) stops runaway spend. If an article changes after its clip was made, the old clip is still played, labelled "earlier version". Regenerating is a separate, explicit action. |
| D11 | **The narration script is built from the article, not the page.** It is the headline, standfirst, byline ("By …, in London") and body paragraphs, with captions, pull quotes, "Recommended"/"Read more" links, subscription prompts and image credits removed. Text over the 60 000-character request limit is split at paragraph boundaries, and the MP3 parts are joined into one file. |
| D12 | **Questions are answered by Hermes, with the same provider, model and key as hermes-trainer.** It is the same pinned Hermes release, run as its own gateway with a `news` profile, so its memory is kept apart from the coach's. Model URL and model id go in `deploy/local.env`. The key is copied root-to-root by `sudo ./deploy/hermes-secrets.sh --from-trainer`, without ever being shown or typed. The profile reaches the store only through a read-only MCP server (`search_articles`, `get_article`, `list_recent`, `list_sections`). Like the coach, it has no terminal, files, web or browser. Answers must cite article ids, and the web app renders the citations as links into the reader. Hermes memory holds the owner's interests and preferences. The article facts stay in SQLite. |
| D13 | **Licensed content never enters the repository.** Article text, subscription headlines, clips, cookies and browser profiles stay on the host. Test fixtures are small hand-written HTML/JSON shaped like the sites' markup, never saved real pages. The policy checker and `.gitignore` back this up. |
| D14 | **Runtime:** the same host as hermes-trainer, under a dedicated `news` system user with its own data directory (mode 0700), its own systemd units and its own `tailscale serve` path. Secrets (the model key, the gateway key, the xAI key) are installed root-only by `deploy/*-secrets.sh` scripts and handed to the services by systemd, as in hermes-trainer. |

### 2.1 Sections collected

The starting list lives in the database and can be changed in settings.
Feeds were checked on 2026-10-04. Phase 2 confirms every entry.

| Topic | FT | Bloomberg |
| --- | --- | --- |
| Markets | `/markets` (RSS, 25 items) | `/feeds/markets/news.rss` (20 items) and the `/markets` page |
| Macro / economics | `/global-economy` (RSS) | `/feeds/economics/news.rss` (5 items) and the `/economics` page |
| Bonds and rates | `/bonds` (RSS) | `/markets/rates-bonds` page |
| Commodities | `/commodities` (RSS) | `/markets/commodities` page |
| Crypto | the crypto section (its old RSS path now redirects; phase 2 finds the new one) | `/crypto` page (its RSS feed is empty) |
| Breaking / latest | the FT news feed | the `/latest` page |
| Broader context | `/world` (RSS) | — |

Expected volume is a few hundred articles a day, which is a few megabytes of
text.

## 3. Architecture

```
iPhone (home-screen PWA)
   │  HTTPS, tailnet only
   ▼
tailscale serve ──▶ newsbot-api (FastAPI, 127.0.0.1)
                      ├─ REST API: feed, reader, listen, ask, source status
                      ├─ /api/clips/{id}: MP3 with HTTP Range (iOS seeking)
                      ├─ SQLite + FTS5 and clip files (private data dir)
                      ├─ Voice service ──▶ xAI TTS (only on "Listen", once per key)
                      └─ Ask proxy ──▶ hermes-gateway (profile: news)
                                         └─ MCP ──▶ newsbot-mcp (read-only tools)

newsbot-fetch.timer ──▶ newsbot fetch (oneshot)
                          ├─ discover: RSS/section feeds (httpx)
                          └─ fetch: Playwright Chromium, persistent profile per source
```

Processes (systemd, all as `news`, loopback only):

1. **newsbot-api**: serves the built web app and the JSON API.
2. **newsbot-fetch** (timer + oneshot): discovery and fetching. It runs as a
   separate process so a hung browser can never block the API. It uses a
   virtual display (`xvfb-run`) so Chromium runs headed with the same profile
   the owner signed in with.
3. **hermes-gateway** (`news` profile) and **newsbot-mcp**: the Q&A path,
   built the same way as hermes-trainer's coach.

Repository layout:

```
src/newsbot/
  domain/    pure models and rules: ArticleDraft, URL canonicalisation, dedup,
             re-fetch policy, narration script, clip key + cache decision, budget
  storage/   SQLite schema, migrations, repositories, FTS5
  sources/   per-source adapters: feed discovery, Playwright session, page parsers
  voice/     xAI TTS client (httpx), MP3 part joining
  services/  use cases: fetch run, read, listen, ask, source health
  api/       FastAPI routers and app factory
  mcp/       Hermes tool surface (read-only)
  cli/       `newsbot login|fetch|migrate|backup`
web/         React + Vite PWA
hermes/      news profile: config.yaml, SOUL.md
deploy/      systemd units, tailscale serve, secrets, backup
docs/        this plan, DEVELOPMENT.md, DEPLOY.md
```

`import-linter` enforces the layers: `api`/`mcp`/`cli` → `services` →
`storage` | `sources` | `voice` (the three adapters cannot import one another)
→ `domain`. `domain` may not import any I/O, browser or web framework.

## 4. Data model (initial)

```
sources        id, slug (ft|bloomberg), name, session_state (ok|needs_attention|signed_out),
               session_checked_at, last_run_at
fetch_runs     id, source_id, started_at, ended_at, discovered, fetched, skipped,
               failed, stop_reason
articles       id, source_id, canonical_url UNIQUE, external_id, headline, standfirst,
               byline, section, published_at, updated_at, fetched_at,
               body_blocks_json, body_text, content_hash, word_count,
               read_at, starred_at
articles_fts   FTS5(headline, standfirst, body_text) — content=articles
clips          id, article_id, content_hash, voice_id, speed, codec, sample_rate,
               bit_rate, file_name, bytes, duration_s, chars, created_at,
               UNIQUE(article_id, content_hash, voice_id, speed, codec, sample_rate, bit_rate)
tts_requests   id, clip_key, article_id, chars, status (running|ok|failed), error,
               started_at, finished_at        -- the audit trail and the daily budget
playback       article_id PRIMARY KEY, position_s, finished_at, updated_at
ask_sessions   id, hermes_session_id, created_at
```

`tts_requests` is what proves the cache works. The phase-3 exit check is
"listen twice, see exactly one row".

## 5. Web app

- **Feed**: newest first, grouped by day, filterable by source and section;
  unread/read and starred; pull to refresh triggers a fetch run.
- **Reader**: headline, standfirst, byline, times, the sanitised body and a
  link to the original. A **Listen** button reads *Play* when a clip exists,
  or *Generate & play (≈ N min, M chars)* when it doesn't.
- **Player**: a mini player that stays open across screens, with play/pause,
  ±15 s, speed 0.75–2× (client-side playback rate, so one clip serves every
  speed), resume from the saved position, and Media Session metadata for
  lock-screen controls.
- **Ask**: chat with Hermes over the article store. Citations open the reader.
  Suggested prompts: "what happened today in …", "compare FT and Bloomberg on …".
- **Sources**: session state per source, last run, failures, and the
  `newsbot login` instruction when a source needs attention. It also shows the
  TTS characters used today against the budget.

## 6. Phases

Each phase is one or more gated pull requests.

| Phase | Scope | Exit criterion |
| --- | --- | --- |
| 0a | **Quality gate**: everything copied from hermes-trainer and adapted (policy checker, lint/type/test/coverage/mutation/dependency/reproducibility gates, CI, CODEOWNERS, Dependabot, branch protection script), a minimal FastAPI app and React shell | All seven checks green on the bootstrap PR; branch protection applied |
| 0b | **Runtime skeleton**: `news` user, data dir, `newsbot-api` unit, `tailscale serve`, deploy + backup scripts (copied from hermes-trainer's `deploy/`) | A health page loads on the phone over the tailnet |
| 1 | **Store and reader**: schema + migrations, FTS5, feed/reader API and screens, `newsbot import-html` for local development against fixtures | Hand-imported fixture articles readable on the phone |
| 2 | **Sources**: Playwright session manager, `newsbot login`, FT then Bloomberg (RSS and section-page discovery, parser, needs-attention detection, 429 back-off), fetch timer, Sources screen | A full day of both sources' articles appears with no manual steps |
| 3 | **Listen**: narration script, clip key + cache + budget rules, xAI client, single-flight generation, Range-served clips, player + resume | An article plays in Zagan. The second play makes zero xAI calls (`tts_requests` has one row) |
| 4 | **Ask**: Hermes gateway with a `news` profile (provider and key shared with hermes-trainer), read-only MCP tools, Ask tab with citations | Correct, cited answers about the day's articles, with context kept across turns |
| 5 | **Polish**: Web Push for "session needs attention", starred reading list, offline caching of starred clips, retention settings | The owner uses it daily without opening the FT or Bloomberg apps |

Later candidates, outside this plan: a morning audio briefing (still needs an
explicit tap to generate), newsletter ingestion from email, more sources behind
the same adapter interface.

## 7. Risks

| Risk | Mitigation |
| --- | --- |
| **Terms of use.** Both publishers' subscriber terms limit automated access, and either could flag or suspend the account. | Personal use only, paced one page at a time (D5); stop at the first sign of a challenge (D6); nothing is redistributed (D13). The owner accepts the remaining risk. |
| **Bot defences** (Bloomberg especially) block headless browsers. | Run headed under Xvfb with the owner's real signed-in profile. Fall back to marking the source needs-attention, never to evading the defence. |
| **Rate limits.** FT answered 429 after six quick RSS requests while this plan was being checked. | One request at a time per site with a minimum gap, back-off on 429, and a different discovery mix per source (D5). |
| **Markup changes** break a parser. | Structured data first. When the parser fails, keep the URL, record the error in `fetch_runs`, show it on the Sources screen, and fix it with a new fixture test. |
| **Playwright on aarch64** (the host is a Raspberry Pi). | Phase 2 starts with a spike: launch Chromium with a persistent profile under Xvfb as `news`. If Playwright's bundled build fails, point it at the distro's Chromium (`executable_path`). |
| **iOS PWA background audio** stops or loses position. | Media Session API, saving the position every 5 s and on `pagehide`, and Range-served MP3 so seeking works. |
| **TTS spend.** | On demand only, at most once per key, a daily character budget, and the `tts_requests` audit trail. |

## 8. Answered questions (2026-10-04)

1. Bloomberg is a **bloomberg.com** digital subscription, not a Terminal.
2. Sections: **macro, markets, crypto, bonds, breaking news and commodities**,
   collected broadly (D4, §2.1).
3. Questions are answered by **Hermes with the same key as hermes-trainer**
   (D12).

Still open, with the default in brackets: how long articles and clips are kept.
[Forever. Text is small, and a clip is about 1 MB per minute at 128 kbps.]

## 9. Owner actions required

- After the gate PR's checks have run once, run
  `scripts/configure_repository.sh` as the repository admin (squash-only
  merges and branch protection on `main`).
- At deploy (phase 4), copy the model URL and model id from hermes-trainer's
  `deploy/local.env` into this repo's, then run
  `sudo ./deploy/hermes-secrets.sh --from-trainer`.
- Create an xAI API key with TTS access (phase 3). It is installed on the host
  by the secrets script and never pasted into chat or the repo.
- Run `newsbot login ft` and `newsbot login bloomberg` once on the host desktop
  (phase 2), and again whenever the Sources screen says so.
